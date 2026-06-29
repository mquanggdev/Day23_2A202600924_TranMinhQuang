# Day 23 Lab Reflection

**Student:** Tran Minh Quang
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/MinhQuangVINUNI/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.5.2)
Compose v2:    OK  (5.1.4)
RAM available: 7.69 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3001, 3100, 16686, 4317, 4318, 8888]
Report written: D:\Python\VINUNI\Day23_2A202600924_TranMinhQuang\00-setup\setup-report.json
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

One thing that surprised me about Prometheus and Grafana is the elegance of the multi-window multi-burn-rate alert math (using both short-window and long-window burn rates to catch sudden high-burn outages quickly while avoiding false alarms from low-burn trends). Additionally, the ability to provision dashboards-as-code from raw JSON definitions in the `provisioning` directory makes it extremely easy to reproduce environments without manual GUI configurations.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1594, "trace_id": "5e8b3631f53e2ff2562037871af5f280", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T09:56:00.359572Z"}
```
Trace ID: `5e8b3631f53e2ff2562037871af5f280`

### Tail-sampling math

In `otel-config.yaml`, the tail-sampling processor is configured with three policies:
1. `keep-errors`: keeps 100% of traces with status ERROR.
2. `keep-slow`: keeps 100% of traces with latency > 2000 ms.
3. `probabilistic-1pct`: keeps 1% (0.01 fraction) of all remaining healthy/normal traces.

Let $N$ be the total traces per second, $E$ be the number of error traces per second, $S$ be the number of slow traces per second (latency > 2000ms), and $H$ be the healthy/normal traces per second (where $H = N - E - S$).

The total number of traces retained per second is:
$$\text{Retained Traces} = E + S + 0.01 \times H$$

So the fraction kept is:
$$\text{Kept Fraction} = \frac{E + S + 0.01 \times (N - E - S)}{N}$$

If all traces are healthy and fast ($E=0, S=0$), then the kept fraction is exactly $1\%$ (0.01). If a failure spike occurs where 20% of requests fail ($E = 0.2N$), then the collector will retain at least $20.8\%$ of all traces, capturing all error paths while keeping normal tracing volume extremely low.

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
    "drift": "no"
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

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why:
- `prompt_length`: **KS Test** or **PSI**. Prompt length is a discrete continuous numerical value (integer). KS test is a great fit to compare continuous cumulative distributions without binning bias. Alternatively, PSI is useful if we want to bin prompt lengths (e.g. <50 tokens, 50-200, >200) to match specific pricing/latency brackets for business operations.
- `embedding_norm`: **KS Test**. The norm of an embedding vector is a continuous, single-dimensional floating-point value reflecting vector magnitude. The Kolmogorov-Smirnov test is highly sensitive to shifts in the shape, spread, or location of continuous distributions without requiring binning.
- `response_length`: **KS Test** or **PSI** (similar reasoning to prompt_length).
- `response_quality`: **PSI** or **KS Test**. Response quality is a continuous score (usually [0,1]). Since quality is often used for SLA and compliance reporting, binning it using PSI (e.g., poor [0, 0.5), acceptable [0.5, 0.8), high [0.8, 1]) is highly interpretable for product alerts. For mathematical precision, the KS test is also excellent.
- *For high-dimensional raw embedding vectors*: **MMD (Maximum Mean Discrepancy)**. For raw high-dimensional embeddings (e.g. 768-dim vectors), simple univariate tests like KS or PSI cannot be applied directly without losing covariance. MMD is a kernel-based test designed specifically to detect shifts in multivariate/high-dimensional spaces.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Exposing metrics from the model-serving layer (llama.cpp) was the most challenging. Unlike database engines or cloud infrastructure hosts, llama.cpp does not natively expose a Prometheus endpoint directly. We had to implement a sidecar container to query its active slot status via HTTP APIs and translate it to Prometheus-scrapeable metric formats. This requires coordinating trace propagation across the API and serving boundaries to avoid losing context.

---

## 6. The single change that mattered most

The single design change that made the biggest difference was adding tail-sampling logic to the OpenTelemetry Collector and correlating it with structured JSON application logs. In high-throughput AI services, logging or tracing 100% of requests is financially and operationally prohibitive due to network/storage overhead (finops constraints). By keeping only 1% of normal healthy spans but 100% of errors and high-latency traces, we kept the observability bill manageable while ensuring that we never miss a critical failure mode or bottleneck. Correlating this with `trace_id` injection in JSON logs allows developers to move seamlessly from a high-level Grafana dashboard to Loki logs, and finally to a detailed Jaeger trace, reducing the Mean Time to Resolution (MTTR) dramatically during an incident. This directly implements the observability flywheel from Section 14 of the lecture deck.

---

## 7. Bonus Track B3 — AgentOps Reflection

### Why $pass^k \neq pass@k$ is important for my agent:
$pass@k$ is a static, offline evaluation metric that estimates the probability that at least one of $k$ independent generation runs passes a test suite. However, an active agent operates in a trajectory loop ($pass^k$) where it reflects, handles tool errors, corrects its plan, and tries again up to $k$ steps. Therefore, agent success depends on its dynamic error-recovery and loop-detection capabilities during execution, rather than just independent random draws. Observing this online trajectory (e.g., measuring `loops_detected` or `avg_steps_per_task`) is crucial for understanding agent reliability in production.

### First SLI to Alert On:
I would alert first on `loops_detected` or `avg_steps_per_task` exceeding a safety threshold. An agent caught in a loop or running too many steps without reaching its goal is actively consuming LLM tokens (costing money) and degrading latency without delivering value, indicating a breakdown in the agent's reasoning capability.
