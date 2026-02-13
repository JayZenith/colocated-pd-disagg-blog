# Prefill/Decode Disaggregation on 4× RTX 3090 (PCIe): A Simple A/B

I wanted to answer one practical question:

**On a single 4-GPU PCIe machine, does prefill/decode (PD) disaggregation beat a colocated engine?**

This is *not* a “PD is good/bad in general” post, just a grounded single-node result.

code: [github.com/JayZenith/pd-disagg](https://github.com/JayZenith/pd-disagg)


---

## Setup

- **Hardware:** 4× RTX 3090 (24GB), PCIe (single machine)
- **Model:** DeepSeek-V2-Lite-Chat (~15.7B, bf16)
- **Serving:** Ray Serve LLM + vLLM
- **KV transfer (PD only):** NixlConnector
- **gpu_memory_utilization:** 0.97 on both configs

### Configs

**Colocated**
- 1 engine
- **TP=4** (all 4 GPUs)

**Disaggregated (PD)**
- Prefill engine: **TP=2**
- Decode engine: **TP=2**
- Prefill produces KV → decode consumes KV (via NIXL)

---

## Workload + measurement

- **Non-streaming only**
- **max_tokens = 128**
- Short fixed prompt set
- Measured with Locust:
  - **RPS** = requests/sec
  - **p50/p95 latency** = end-to-end response time (ms)

> Important: these latencies are end-to-end request times (queueing + compute + overhead).

---

## Results

From the Locust stats CSV (one run per concurrency):

| Concurrency | Colocated RPS | Coloc p50 (ms) | Coloc p95 (ms) | PD RPS | PD p50 (ms) | PD p95 (ms) | PD Δ RPS |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 1  | 0.068 | 14000 | 14000 | 0.088 | 14000 | 14000 | +29.3% |
| 4  | 0.352 | 12000 | 13000 | 0.355 | 12000 | 15000 | +0.7% |
| 8  | 0.615 | 13000 | 13000 | 0.662 | 12000 | 12000 | +7.6% |
| 16 | 1.230 | 13000 | 13000 | 1.268 | 12000 | 12000 | +3.1% |

**Read this like an engineer:**
- At **c=1**, there were only **4 total requests** in 60s, so don’t over-interpret the +29%.
- At **c=8–16** (the only part I actually care about), **PD is slightly ahead in RPS** in this run, and p95 is in the same ballpark (~12–13s).
- At **c=4**, throughput is basically even, and PD p95 was worse in this run (15s vs 13s).

So the honest summary is:

> On this single-node PCIe setup and fixed 128-token workload, PD (TP=2/2) is **near-parity** with colocated TP=4, sometimes slightly better in throughput, with similar tail latency.

---

## Constraints / what didn’t work

I tried a decode-heavier split (prefill TP=1, decode TP=3) and it failed:
- **TP=3 is invalid** for this model because attention heads aren’t divisible by 3.
- **TP=1 prefill OOM’d** on a single 3090 for this model/config.

So TP=2/2 is the realistic PD split for this machine.

---

## What this does (and doesn’t) prove

**This does not prove** “PD is always good” or “PD is always bad.”  
It shows something narrower:

- On a single 4×3090 PCIe node, PD disaggregation is **not automatically a win**, but it can be **competitive** for fixed-length generation.
- The “big PD advantages” likely require a different regime:
  - long prompts (prefill is expensive)
  - mixed prompt lengths (prefill interference is real)
  - multi-node / hetero hardware where you can scale prefill and decode separately

---

## What I’d test next (if I extend this)

1) **Mixed prompt lengths** (short + very long prompts) to see if PD helps tail latency under interference  
2) **Long prompt / RAG-like inputs** (prefill-heavy workloads)  
3) **Multi-node** PD (where separation/scaling is actually the point)

---

## Repro (high level)

- Run colocated app (TP=4)
- Run PD app (prefill TP=2, decode TP=2)
- Run Locust at concurrencies 1/4/8/16 for 60s each
- Compare RPS + p50/p95 from Locust stats

