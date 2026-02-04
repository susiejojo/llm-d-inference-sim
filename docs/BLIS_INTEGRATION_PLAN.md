# BLIS Integration Plan

## Objective

Integrate BLIS's full scheduling simulation into llm-d-inference-sim with minimal code changes.

---

## Approach

Hide BLIS behind the existing `LatencyCalculator` interface. All complexity stays inside the new `blisCalculator` implementation.

---

## Architecture

```
llm-d-inference-sim (unchanged)
         │
         │ calls GetTimeToFirstToken() / GetInterTokenLatency()
         ▼
┌─────────────────────────────────────────────┐
│            blisCalculator                   │
│  (implements LatencyCalculator interface)   │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │  Internal: channels + goroutine     │    │
│  │                                     │    │
│  │  arrivalChan ──► BLIS Controller ◄──│    │
│  │  resultChan  ◄── (goroutine)        │    │
│  │                      │              │    │
│  │                      ▼              │    │
│  │              BLIS Simulator         │    │
│  │         (WaitQ, Batch, KVCache)     │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

---

## How It Works

1. **Request arrives** → `GetTimeToFirstToken()` called
2. **blisCalculator** sends arrival to BLIS controller via channel
3. **BLIS controller** injects into simulator, runs `Step()`, returns prediction
4. **blisCalculator** returns duration to llm-d-inference-sim
5. **llm-d-inference-sim sleeps** (unchanged behavior)
6. **Next call** to `GetInterTokenLatency()` triggers next step

---

## Goroutines & Concurrency

```
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    Request 1    │  │    Request 2    │  │    Request 3    │
│   (goroutine)   │  │   (goroutine)   │  │   (goroutine)   │
│       │         │  │       │         │  │       │         │
└───────┼─────────┘  └───────┼─────────┘  └───────┼─────────┘
        │                    │                    │
        └───────────────────►├◄───────────────────┘
                             │
                        arrivalChan
                             │
                             ▼
               ┌─────────────────────────┐
               │    BLIS Controller      │
               │    (single goroutine)   │
               │                         │
               │  ┌───────────────────┐  │
               │  │  BLIS Simulator   │  │
               │  │  (WaitQ, Batch,   │  │
               │  │   KVCache, etc.)  │  │
               │  └───────────────────┘  │
               └─────────────────────────┘
                             │
                        resultChans
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ req1 gets     │  │ req2 gets     │  │ req3 gets     │
│ TTFT, sleeps  │  │ TTFT, sleeps  │  │ TTFT, sleeps  │
└───────────────┘  └───────────────┘  └───────────────┘
```

Each incoming request runs in its own goroutine (Go does this automatically). All requests send to one BLIS controller, which serializes access to the simulator.

---

## Changes Required

### llm-d-inference-sim (minimal)

| File | Change |
|------|--------|
| `go.mod` | Add BLIS dependency |
| `context.go` | Add `case "blis": return newBlisCalculator(cfg)` |
| `config.go` | Add BLIS config params |
| `blis_calculator.go` | New file: implements `LatencyCalculator` with internal BLIS controller |

### BLIS (inference-sim)

| Change | Purpose |
|--------|---------|
| `InjectArrival()` | Add request to WaitQ |
| `RunOneStep()` | Run one scheduler step, return results |
| Skip workload pre-generation | Arrivals come from external calls |

---

## Configuration

```yaml
latency-calculator: "blis"
model: "meta-llama/llama-3.1-8b-instruct"
hardware: "H100"
tensor-parallelism: 1
model-config-path: "inference-sim/model_configs/llama-3.1-8b-instruct/config.json"
hardware-config-path: "inference-sim/hardware_config.json"
```

---

## Summary

- **llm-d-inference-sim**: No changes to request handling flow
- **BLIS**: Full scheduling simulation preserved
- **Integration**: Hidden behind existing interface via channels + controller goroutine
