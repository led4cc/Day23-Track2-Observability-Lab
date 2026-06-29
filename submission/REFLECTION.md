# Day 23 Lab Reflection

**Student:** Nghie
**Submission date:** 2026-06-30
**Lab repo URL:** _add public GitHub URL before submission_

---

## 1. Hardware + setup output

Output/artifact from `python3 00-setup/verify-docker.py`:

```json
{
  "docker": {
    "ok": true,
    "version": "29.2.1"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.0.2"
  },
  "ram_gb_available": 7.63,
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

The stack ran in Docker Compose with the FastAPI app, Prometheus, Grafana, Alertmanager, Loki, Jaeger, and OpenTelemetry Collector. `make smoke` passed after fixing local readiness/config issues.

---

## 2. Track 02 - Dashboards & Alerts

### 6 essential panels

Evidence: `submission/screenshots/dashboard-overview.png`

The AI Service Overview dashboard showed request rate, latency percentiles, GPU utilization, token throughput, and in-flight requests after running Locust load against `/predict`.

### Cost and tokens

Evidence: `submission/screenshots/cost-and-tokens.png`

The Cost & Tokens dashboard showed input/output token throughput, estimated hourly cost, and the latest eval quality score for `llama3-mock`.

### Burn-rate panel

Evidence: `submission/screenshots/slo-burn-rate.png`

I generated error traffic with `ERROR_RATE=0.2` in Locust. This produced visible burn-rate curves and an active SLO burn alert.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | Killed `day23-app` with `make alert` | `submission/screenshots/alertmanager-firing.png` |
| T0+90s | `ServiceDown` fired in Prometheus | `submission/screenshots/alertmanager-firing.png` |
| T1 | App restored by script | terminal output from `make alert` |
| T1+60s | Alert resolved | terminal output showed `alert resolved` |

Slack webhook was not committed to the public repo; local alert evidence was captured through Prometheus/Alertmanager. The placeholder Slack URL should be replaced only locally if Slack screenshots are required, then reverted before pushing.

### One thing surprised me about Prometheus / Grafana

The biggest surprise was how easy it is for dashboards to look "empty" even when Prometheus is scraping correctly. In this lab, the app metrics existed, but the Grafana datasource UID did not match the dashboard JSON until I fixed the provisioned UID to `prometheus`. I also saw that `rate(...[1m])` can show no useful graph after load stops, so the time window and refresh timing matter a lot.

---

## 3. Track 03 - Tracing & Logs

### One trace screenshot from Jaeger

Evidence: `submission/screenshots/jaeger-trace.png`

The retained Jaeger trace used trace ID:

```text
5a10c1ff32a1f1874c32b045df2f52ce
```

It showed these spans in one trace:

```text
predict
embed-text
vector-search
generate-tokens
```

### GenAI span attributes

Evidence: `submission/screenshots/jaeger-genai-attrs.png`

The `generate-tokens` span included:

```text
gen_ai.usage.input_tokens = 4
gen_ai.usage.output_tokens = 27
gen_ai.response.finish_reason = stop
```

The `predict` span included:

```text
gen_ai.request.model = llama3-mock
```

### Log line correlated to trace

Structured JSON log line linked to the retained trace:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 27, "quality": 0.827, "duration_seconds": 2.0514, "trace_id": "5a10c1ff32a1f1874c32b045df2f52ce", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T16:38:58.963109Z"}
```

### Tail-sampling math

The OTel Collector policy keeps:

```text
100% of ERROR traces
100% of slow traces above 2000 ms
1% of healthy traces
```

For the normal Locust load, the dashboard peaked around 18 requests/sec. If all of those requests are healthy, the probabilistic policy keeps about:

```text
18 traces/sec * 0.01 = 0.18 traces/sec
```

That is why many healthy traces did not appear in Jaeger. To capture a reliable trace for evidence, I used a deterministic slow prompt that produced a trace above 2 seconds, which matched the `keep-slow` policy and was retained.

---

## 4. Track 04 - Drift Detection

### PSI scores

Contents of `04-drift-detection/reports/drift-summary.json`:

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

Evidence: `submission/screenshots/drift-report.png`

### Which test fits which feature?

For `prompt_length`, I would use PSI as the first production monitor because prompt length is a binned distribution and PSI is easy to explain to operators. I would keep KS as a secondary check because it detects distribution shifts without requiring a specific binning choice.

For `embedding_norm`, I would use KS or KL. It is continuous and should be relatively stable, so KS is a good simple detector for distribution changes; KL is useful when I want to quantify how far the current distribution has moved from baseline.

For `response_length`, I would use PSI plus KS. Response length is operationally understandable in buckets, but KS is useful because length is still numeric and can drift subtly before bucketed PSI becomes obvious.

For `response_quality`, I would use KS and PSI together. Quality score is continuous and model-facing, so KS catches score distribution shifts directly, while PSI gives a dashboard-friendly signal for stakeholders. In this run, `response_quality` drifted strongly, so it would be a high-priority investigation target.

---

## 5. Track 05 - Cross-Day Integration

Evidence: `submission/screenshots/cross-day-dashboard.png`

The cross-day dashboard rendered all six panels for Days 16, 17, 18, 19, 20, and 22. I did not have the prior-day systems running locally, so the panels intentionally showed `No Data` instead of breaking.

### Which prior-day metric was hardest to expose? Why?

The hardest prior-day metric would likely be Day 20 llama.cpp tokens/sec or Day 19 Qdrant metrics. Both require an external service from a previous lab to be running at a stable address from inside Docker, usually through `host.docker.internal`, and the scrape target must expose Prometheus-format metrics. That is harder than a local app metric because the observability stack depends on another lab's network, port, and exporter behavior being correct.

---

## 6. The single change that mattered most

The single change that mattered most was making the metric and trace labels stable and low-cardinality: `model`, `status`, and `direction` were enough to explain user-visible behavior without exploding the number of Prometheus series. With those labels, the same raw signals powered the overview dashboard, cost dashboard, SLO burn-rate rules, and the debugging path into Jaeger. Dropping unnecessary labels is what made the stack feel usable instead of merely noisy.

This connects directly to the deck's observability and SLO sections: the goal is not to collect every possible signal, but to keep the signals that let an operator move from symptom to cause. `status` made the SLO math possible, `direction` made token cost understandable, and the GenAI trace attributes made a single request explainable. That small label design choice was the difference between "the app emits metrics" and "the stack can answer what changed, how bad it is, and where to look next."
