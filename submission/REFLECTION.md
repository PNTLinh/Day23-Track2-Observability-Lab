# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Thùy LInh
**Submission date:** _2026-05-11_
**Lab repo URL:** https://github.com/PNTLinh/Day23-Track2-Observability-Lab.git

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:
```
{
  "docker": {
    "ok": true,
    "version": "29.4.0"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.1"
  },
  "ram_gb_available": 5.46,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [],
  "all_ports_free": true
}
```
---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

_(2-3 sentences)_

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans with parent span `predict` wrapping 3 child operations.

**Trace Evidence:** 
- Trace ID: `8a4530bf2391e5d7d072345ccaa9fe2c`
- Span count: 4 (1 parent + 3 children)
- Operations: predict (parent), embed-text, vector-search, generate-tokens
- Latency: 2.2s (kept 100% by tail-sampling policy because > 2000ms threshold)

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 8, "output_tokens": 8, "quality": 0.855, "duration_seconds": 0.1652, "trace_id": "9272494df403b66e7469d77b8e42aa2e", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T08:54:41.843284Z"}
```

Link in Grafana Loki → Jaeger via `trace_id` derived field: Click `trace_id` value → auto-navigates to `http://localhost:16686/trace/9272494df403b66e7469d77b8e42aa2e`.

### Tail-sampling math

Generated **~350 traces** during load test. OTel Collector tail-sampling policy (decision_wait: 30s):
- **100% of errors** (status_code != 0) → ~5 traces kept (error rate ~1-2%)
- **100% of slow** (latency > 2000ms) → ~40 traces kept (~11% of requests)
- **1% of healthy** (status_code = 0, latency < 2000ms) → ~3 traces kept (~1% of ~300 healthy)
- **Total kept:** ~48 / 350 = **13.7%** sampled to Jaeger (rest discarded by policy)

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
**Day 19 (Vector Store)** would be hardest: embeddings are high-dimensional (384+D) and expensive to log at scale; most observability stacks avoid storing full embeddings in logs/traces. Instead, only the embedding_norm (L2 magnitude) is tractable to monitor — but loses information about which semantic dimensions drifted. This is why Day 23 focuses on drift detection: the vector store itself is opaque to metrics; drift testing is the post-hoc inference tool.

**Day 20 (Serving)** would be second-hardest: model serving latency is sensitive to batch size, GPU state, and request order — difficult to attribute to code vs. infra without detailed profiling (see BONUS-ebpf-profiling). Standard metrics like `/metrics` endpoints can't capture this nuance.
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

- **prompt_length** → **PSI (Population Stability Index)**: Detects shifts in categorical/discrete distributions. Prompt length is histogram-like; PSI=3.461 confirms distribution shift (users now sending longer prompts).

- **embedding_norm** → **KS (Kolmogorov-Smirnov)**: Best for continuous numeric features; tests whether CDF shapes differ. embedding_norm shows no drift (KS=0.052, p=0.134), meaning embedding magnitudes remain stable.

- **response_length** → **KL (Kullback-Leibler divergence)**: Asymmetric divergence measuring relative entropy; ideal for NLP token counts. response_length no drift (KL=0.018) → model output length stable despite input changes.

- **response_quality** → **PSI + KL together**: Quality is binned/scaled; PSI=8.849 + KL=13.501 both flag drift (worst case). Suggests model quality perception shifted significantly — likely due to prompt_length shift pulling model into unfamiliar regime.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

_(2-3 sentences. If you didn't have prior days running, write about which one would be hardest based on the integration scripts.)_

---

## 6. The single change that mattered most

### Manual span hierarchy in OTel (predict parent + 3 child ops)

The single most impactful change was restructuring the FastAPI `/predict` endpoint to emit a **parent `predict` span wrapping three child spans** (`embed-text`, `vector-search`, `generate-tokens`) instead of auto-instrumentation alone. This directly connects to **Deck §6 (Tracing + Sampling)**: the span tree reveals the **critical path** — which sub-operations dominate latency? With flat spans, a 2.2s request shows as four isolated operations; with nesting, it becomes immediately clear that `vector-search` (10ms) is not the bottleneck compared to the 5ms + 10ms + unaccounted overhead.

This hierarchy enabled the tail-sampling policy to function meaningfully: keeping only "interesting" traces (>2s or errors) compresses the observability surface from 350→48 traces (~13.7%) while preserving the slowest requests — exactly the ones operators need to debug. Without nesting, sampling stats would be meaningless noise. This reflects the observability principle from Deck §2 (Three Pillars): **metrics tell you there's a problem, traces tell you what the problem is, logs tell you why** — but only if traces are structured (hierarchical) enough to answer "why is this path slow?"

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.
