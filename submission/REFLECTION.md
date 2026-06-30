# Day 23 Lab Reflection

**Student:** _your name_
**Submission date:** 2026-06-29
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Output from `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.1.3"
  },
  "compose_v2": {
    "ok": true,
    "version": "2.40.3+ds1-0ubuntu1"
  },
  "ram_gb_available": 14.82,
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

### 6 essential panels

Screenshot: `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Screenshot: `submission/screenshots/slo-burn-rate.png`.

### Cost and tokens

Screenshot: `submission/screenshots/cost-and-tokens.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| 2026-06-29 22:06 ICT | stopped `day23-app` | `submission/screenshots/alertmanager-firing.png` |
| 2026-06-29 22:08 ICT | `ServiceDown` fired after rule dwell/evaluation | `submission/screenshots/alertmanager-firing.png` |
| 2026-06-29 22:08 ICT | restored `day23-app` | command output from alert drill |
| 2026-06-29 22:09 ICT | alert resolved | `submission/screenshots/alertmanager-resolved.png` |

Slack webhook was configured on 2026-06-30 and a direct test message was accepted by Slack. I then reran `make alert`; `ServiceDown` fired after 18 polling intervals, the app was restored, and the alert resolved. Slack fire/resolve messages should be captured as `submission/screenshots/slack-firing.png` and `submission/screenshots/slack-resolved.png` from the selected Slack channel.

### One thing surprised me about Prometheus / Grafana

The biggest surprise was that dashboard-as-code depends on stable datasource UIDs, not just datasource names. The panels were correctly written, but Grafana initially provisioned Prometheus with a generated UID, so the dashboards rendered with query errors until I pinned the Prometheus datasource UID to `prometheus`.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Screenshot: `submission/screenshots/jaeger-trace.png`.

Trace retained by the slow-trace policy:

```text
trace_id: 9de5160776765ae3046773ed0c123bad
span_count: 4
operations: vector-search, generate-tokens, embed-text, predict
```

### Log line correlated to trace

Structured JSON log line with `trace_id` and `span_id`:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 10, "quality": 0.884, "duration_seconds": 0.1679, "trace_id": "9aa0c6c9b714bf513b2fb6f46c8b4720", "span_id": "6a72e525efbf7159", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T15:05:14.865464Z"}
```

### Tail-sampling math

The collector policy keeps 100% of error traces, 100% of slow traces over 2s, and 1% of healthy traces. During the load test the service produced about 19.13 traces/sec. If traffic is roughly 1% error, 1% slow, and 98% healthy, then:

```text
sampled = 19.13 * (0.01 + 0.01 + 0.98 * 0.01)
        = 19.13 * 0.0298
        = 0.57 traces/sec retained
```

That is about 3% trace retention, or roughly 97% less trace storage than keeping every healthy request. For the evidence trace I used a deterministic prompt that produced a >2s span, so the `keep-slow` policy retained it.

---

## 4. Track 04 — Drift Detection

### PSI scores

Screenshot: `submission/screenshots/drift-report.png`.

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

### Which test fits which feature?

For `prompt_length`, I would use PSI for dashboarding and KS as a statistical confirmation. Prompt length is continuous but easy to bucket, and PSI is intuitive for explaining "production prompts are longer than baseline."

For `embedding_norm`, I would use KS for the scalar norm and MMD if comparing full embedding vectors. A single norm can miss directional embedding shift, while MMD is better for multivariate representation drift.

For `response_length`, I would use PSI or KS. The operational question is whether outputs became much shorter or longer, so a bucketed PSI trend is useful for alerting and a KS p-value is useful for confirming the distribution shift.

For `response_quality`, I would use KS plus PSI. Quality is bounded and continuous, and in this run both PSI and KS caught a severe degradation from high-quality baseline responses to lower current quality.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Screenshot: `submission/screenshots/cross-day-dashboard.png`.

The hardest prior-day metric would be Day 20 model serving cost/performance, because it depends on whether the model server exposes Prometheus metrics and whether GPU/token counters are standardized. Qdrant-style vector store metrics are usually scrapeable directly, but LLM serving often needs normalization across llama.cpp, vLLM, or a hosted API before the dashboard can compare latency, tokens/sec, and cost.

---

## 6. The single change that mattered most

The single change that mattered most was pinning the telemetry contracts: stable Grafana datasource UIDs and a correct trace parent/child relationship in the FastAPI instrumentation. Without those, the system technically emitted signals, but the dashboards showed query errors and Jaeger showed isolated spans instead of the real request trajectory. After fixing the datasource UID and making `predict` the parent span, the stack became useful: a dashboard can answer "is the service healthy?" and a trace can answer "where did this request spend time?"

This connects directly to the deck's RED/USE plus tracing sections. Metrics tell me the request rate, errors, duration, GPU utilization, tokens, and quality changed; traces tell me whether the latency came from embedding, vector search, or generation. That bridge is what turns observability from "many tools are running" into an operational loop where an on-call engineer can detect, localize, and explain the issue quickly.
