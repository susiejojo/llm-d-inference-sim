# BLIS Integration Plan

## Objective

Integrate BLIS into llm-d-inference-sim as a Go library, reusing BLIS's existing `Simulator` struct and per-request metrics.

## Key Insight: BLIS Already Captures Per-Request Latencies

BLIS's `Simulator` (`sim/simulator.go`) already tracks per-request timing in its `Metrics` struct:

```go
sim.Metrics.RequestTTFTs[req.ID]    // Per-request TTFT (line 447)
sim.Metrics.RequestITLs[req.ID]     // Per-request ITL average (line 498)
sim.Metrics.RequestE2Es[req.ID]     // Per-request E2E latency (line 493)
sim.Metrics.AllITLs                 // All individual ITL values (line 505)
```

We just need to expose these for library usage.

---

## Step 1: Export BLIS Simulator for Library Use

**Goal**: Make existing BLIS types accessible as a Go package.

The `Simulator` struct is already exported (capital S). We need to:

1. **Export `Metrics` fields** (if not already):
   ```go
   // sim/metrics.go - ensure these are exported
   type Metrics struct {
       RequestTTFTs    map[string]float64  // Already exists
       RequestITLs     map[string]float64  // Already exists
       RequestE2Es     map[string]float64  // Already exists
       AllITLs         []int64             // Already exists
       // ...
   }
   ```

2. **Add helper method to get results** (`sim/simulator.go`):
   ```go
   // GetRequestResults returns per-request latencies after simulation
   func (sim *Simulator) GetRequestResults() map[string]RequestResult {
       results := make(map[string]RequestResult)
       for id, ttft := range sim.Metrics.RequestTTFTs {
           results[id] = RequestResult{
               TTFTMs: ttft / 1000.0,  // Convert microseconds to ms
               ITLMs:  sim.Metrics.RequestITLs[id] / 1000.0,
               E2EMs:  sim.Metrics.RequestE2Es[id] / 1000.0,
           }
       }
       return results
   }

   type RequestResult struct {
       TTFTMs float64
       ITLMs  float64
       E2EMs  float64
   }
   ```

**Files to modify**:
- `inference-sim/sim/simulator.go` - Add `GetRequestResults()` helper
- `inference-sim/sim/metrics.go` - Ensure fields are exported

---

## Step 2: Add Single-Request Simulation Mode

**Goal**: Allow simulating one request at a time for real-time latency prediction.

Currently `NewSimulator()` generates workload upfront. Add mode for on-demand requests:

```go
// sim/simulator.go

// SimulateSingleRequest predicts latencies for a single request given current state
func (sim *Simulator) SimulateSingleRequest(promptTokens, outputTokens int) RequestResult {
    // Create request with given parameters
    req := &Request{
        ID:           fmt.Sprintf("req-%d", time.Now().UnixNano()),
        InputTokens:  generateTokens(promptTokens),
        OutputTokens: generateTokens(outputTokens),
        ArrivalTime:  sim.Clock,
        State:        "queued",
    }

    // Enqueue and run until this request completes
    sim.EnqueueRequest(req)

    // Run simulation steps until request completes
    for req.State != "completed" && len(sim.EventQueue) > 0 {
        ev := heap.Pop(&sim.EventQueue).(Event)
        sim.Clock = ev.Timestamp()
        ev.Execute(sim)
    }

    return RequestResult{
        TTFTMs: sim.Metrics.RequestTTFTs[req.ID] / 1000.0,
        ITLMs:  sim.Metrics.RequestITLs[req.ID] / 1000.0,
        E2EMs:  sim.Metrics.RequestE2Es[req.ID] / 1000.0,
    }
}
```

**Files to modify**:
- `inference-sim/sim/simulator.go` - Add `SimulateSingleRequest()`

---

## Step 3: Integrate into llm-d-inference-sim

**Goal**: Create latency calculator that wraps BLIS `Simulator`.

**Add to `go.mod`**:
```
require github.com/inference-sim/inference-sim v0.0.0
replace github.com/inference-sim/inference-sim => ./inference-sim
```

**New calculator** (`pkg/llm-d-inference-sim/latencies.go`):
```go
import (
    "sync"
    blissim "github.com/inference-sim/inference-sim/sim"
)

type blisCalculator struct {
    simulator *blissim.Simulator
    mu        sync.Mutex
}

func newBlisCalculator(cfg *common.Config) (*blisCalculator, error) {
    // Reuse BLIS's existing NewSimulator
    sim := blissim.NewSimulator(
        cfg.Horizon,
        cfg.Seed,
        cfg.TotalKVBlocks,
        cfg.BlockSizeTokens,
        int64(cfg.MaxNumSeqs),
        int64(cfg.MaxScheduledTokens),
        cfg.LongPrefillTokenThreshold,
        cfg.BetaCoeffs,
        cfg.AlphaCoeffs,
        nil,  // GuideLLMConfig - not needed for single requests
        cfg.ModelConfig,
        cfg.HWConfig,
        cfg.Model,
        cfg.Hardware,
        cfg.TensorParallelism,
        cfg.UseRoofline,
        "",  // No traces file
    )
    return &blisCalculator{simulator: sim}, nil
}

func (b *blisCalculator) GetTimeToFirstToken(params *TTFTParams) time.Duration {
    b.mu.Lock()
    defer b.mu.Unlock()

    result := b.simulator.SimulateSingleRequest(params.PromptTokens, 1)
    return time.Duration(result.TTFTMs * float64(time.Millisecond))
}

func (b *blisCalculator) GetInterTokenLatency(params *InterTokenParams) time.Duration {
    b.mu.Lock()
    defer b.mu.Unlock()

    // Use running batch state to estimate ITL under current load
    batchSize := len(b.simulator.RunningBatch.Requests)
    result := b.simulator.SimulateSingleRequest(params.ContextLength, 1)
    return time.Duration(result.ITLMs * float64(time.Millisecond))
}
```

**Files to modify**:
- `go.mod` - Add BLIS dependency
- `pkg/llm-d-inference-sim/latencies.go` - Add `blisCalculator`
- `pkg/llm-d-inference-sim/context.go` - Register calculator
- `pkg/common/config.go` - Add BLIS-specific config params

---

## Reused BLIS Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `Simulator` | `sim/simulator.go:72` | Main simulation state |
| `NewSimulator()` | `sim/simulator.go:115` | Constructor with all params |
| `Metrics` | `sim/metrics.go` | Per-request latency tracking |
| `KVCache` | `sim/kv_cache.go` | Memory pressure modeling |
| `WaitQ` | `sim/simulator.go:78` | Request queue state |
| `RunningBatch` | `sim/simulator.go:85` | Active batch state |
| `Step()` | `sim/simulator.go:403` | Single simulation step |
| `rooflineStepTime()` | `sim/roofline_step.go:143` | Roofline latency model |

---

## Configuration

```yaml
latency-calculator: "blis"

# BLIS simulation parameters (passed to NewSimulator)
model: "meta-llama/llama-3.1-8b-instruct"
hardware: "H100"
tensor-parallelism: 1
use-roofline: true

# vLLM scheduling knobs
max-num-seqs: 256
max-scheduled-tokens: 2048
total-kv-blocks: 10000
block-size-tokens: 16

# Optional: paths to config files
model-config-path: "inference-sim/model_configs/llama-3.1-8b-instruct/config.json"
hardware-config-path: "inference-sim/hardware_config.json"
```

---

## Summary

| Step | Goal | Key Changes |
|------|------|-------------|
| 1 | Export results | Add `GetRequestResults()` helper to existing `Simulator` |
| 2 | Single-request mode | Add `SimulateSingleRequest()` method |
| 3 | Integration | `blisCalculator` wrapping existing `Simulator` |

This approach maximizes reuse of BLIS's existing code - we're adding ~50 lines to BLIS rather than creating parallel structures.
