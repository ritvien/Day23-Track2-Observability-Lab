# Day 23 Lab Reflection

**Student:** Nguyễn Việt Hoàng
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/ritvien/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Output from `python 00-setup/verify-docker.py`:

```json
{
  "docker": {"ok": true, "version": "29.5.3"},
  "compose_v2": {"ok": true, "version": "5.1.4"},
  "ram_gb_available": 7.65,
  "ram_ok": true,
  "required_ports": [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 - Dashboards & Alerts

Grafana provisioned the Day 23 dashboards automatically:

- `AI Service Overview (Day 23)`
- `SLO Burn Rate (Day 23)`
- `Cost & Tokens (Day 23)`
- `Cross-Day Stack (Day 23 integrative)`

I generated load with 10 parallel PowerShell workers for 60 seconds. Prometheus then showed about `28.58` requests/sec and about `1143.97` tokens/sec in the 1 minute window. The active request gauge returned to `0` after the load stopped.

Alert test result:

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | `docker stop day23-app` |
| T0+105s | `ServiceDown` active | Alertmanager `/api/v2/alerts` showed `ServiceDown` with state `active` |
| T1 | restarted app | `docker start day23-app` |
| T1+60s | alert resolved | Alertmanager no longer had active `ServiceDown` |

One thing that surprised me about Prometheus/Grafana: dashboards only became useful after I created traffic with enough cardinality and time range. A dashboard can be technically provisioned but still operationally empty until the service emits representative requests, errors, tokens, and latency buckets.

---

## 3. Track 03 - Tracing & Logs

Jaeger trace used for evidence:

```text
trace_id: 9eac6e5ec71bbb03e7556c2d6dc42072
service: inference-api
spans: predict -> embed-text, vector-search, generate-tokens
```

The `generate-tokens` span included GenAI semantic attributes:

```text
gen_ai.usage.input_tokens = 4
gen_ai.usage.output_tokens = 18
gen_ai.response.finish_reason = stop
```

Structured log line correlated to a request:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 12, "quality": 0.811, "duration_seconds": 0.3403, "trace_id": "e47b98d6c11c75f2c5f480b53b3c6ac7", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T06:25:29.980031Z"}
```

Tail-sampling math: the collector keeps 100% of error traces, 100% of traces with duration over 2 seconds, and only 1% of healthy traces. During testing, normal healthy traces were received by the collector but dropped by the `probabilistic-1pct` policy. I used prompt `slow-tail-candidate-349`, which deterministically produced a `generate-tokens` span around 2.2 seconds, so the `keep-slow` rule retained it for Jaeger.

---

## 4. Track 04 - Drift Detection

`04-drift-detection/reports/drift-summary.json`:

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

Which test fits which feature:

- `prompt_length`: PSI is easy to explain to product and support teams because the distribution shift is a large movement across length buckets. KS is also useful because it catches the CDF shift without assuming normality.
- `embedding_norm`: KS is a good production guard because this should be stable and approximately continuous; small distribution shifts matter more than bucket interpretation.
- `response_length`: PSI is a practical dashboard metric because length changes are histogram-friendly and easy to reason about operationally.
- `response_quality`: KL or PSI both fit because quality is bounded in `[0,1]` and the full shape changed from high-quality beta-like scores to low-quality scores. I would alert on PSI for readability and investigate with KL/KS.

The script also wrote `04-drift-detection/reports/drift-report.html`. Evidently was not installed in this Python 3.9 environment, so the script generated a fallback HTML report from the same PSI/KL/KS results.

---

## 5. Track 05 - Cross-Day Integration

I connected prior-day sources using Docker Compose stub exporters:

```text
day19-stub -> day19_qdrant_collections = 3
day20-stub -> day20_llamacpp_tokens_per_second ~= 22.72
```

The hardest prior-day metric to expose would be Day 20 llama.cpp tokens/sec. Unlike a web API health metric, token throughput needs either native model-server instrumentation or a sidecar that understands request/completion events. Without a stable metric contract, dashboards can show activity but not the true cost and capacity pressure of serving.

---

## 6. The single change that mattered most

The single most important change was fixing the manual tracing context so `predict` became the current span and `embed-text`, `vector-search`, and `generate-tokens` became children in one trace. Before that, the service emitted spans, but they were not useful as an end-to-end story. After the change, one trace showed the actual request path and the slow `generate-tokens` span carried the GenAI token attributes.

That change connects directly to the tracing and sampling concept from the deck: observability is not just collecting signals, it is preserving the relationships between signals. A metric can say latency is high, but a correlated trace can show where the latency lives. Once the trace hierarchy was correct, tail-sampling also became meaningful: healthy traces could be dropped cheaply, while slow traces remained available for debugging.
