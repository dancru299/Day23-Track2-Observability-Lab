# Day 23 Lab Reflection

**Student:** local lab completion by Codex
**Submission date:** 2026-05-11
**Lab repo URL:** local workspace

---

## 1. Hardware + setup output

`python 00-setup/verify-docker.py` was run while the stack was already up, so the required lab ports were correctly reported as bound:

```text
Docker:        OK  (29.3.1)
Compose v2:    OK  (5.1.1)
RAM available: 7.67 GB (OK)
Ports free:    BOUND: [8000, 9090, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: 00-setup/setup-report.json
```

`python scripts/verify.py` passes 12/12 checks after the fixes.

---

## 2. Track 02 - Dashboards & Alerts

Screenshots created:

- `submission/screenshots/dashboard-overview.png`
- `submission/screenshots/slo-burn-rate.png`
- `submission/screenshots/cost-and-tokens.png`
- `submission/screenshots/alertmanager-firing.png`

Alert evidence:

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | Prometheus `up{job="inference-api"}` became 0 |
| T0 + scrape/eval delay | `ServiceDown` fired | `submission/screenshots/alertmanager-firing.png` |
| T1 | restarted `day23-app` | app returned healthy |
| T1 + delay | alert resolved | verified by `python scripts/verify.py` returning 12/12 |

Slack webhook testing was rerun after `.env` was updated. A direct webhook smoke test returned `ok`, then Alertmanager was force-recreated and a real `ServiceDown` fire/resolve drill completed without Slack delivery errors in recent Alertmanager logs. Slack evidence was captured in `submission/screenshots/slack-firing.png` and `submission/screenshots/slack-resolved.png`.

One thing that surprised me: Grafana provisioning can look "done" while the useful part is still wrong if datasource UIDs do not line up with dashboard JSON. Pinning the Prometheus datasource UID to `prometheus` made the dashboards portable instead of relying on Grafana's generated IDs.

---

## 3. Track 03 - Tracing & Logs

Trace screenshot:

- `submission/screenshots/jaeger-trace.png`

The retained trace used:

```text
trace_id: 1d2b8f4f0097bc9c7cd2adba897eb909
spans: predict, embed-text, vector-search, generate-tokens
duration: 2.19s
```

Log line correlated to trace:

```json
{"model": "llama3-mock", "input_tokens": 5, "output_tokens": 30, "quality": 0.939, "duration_seconds": 2.1923, "trace_id": "1d2b8f4f0097bc9c7cd2adba897eb909", "span_id": "86409acae8749832", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T03:08:50.823983Z"}
```

Loki is reachable, but this repo does not include Promtail or an OTel filelog receiver mounted to Docker stdout, so the correlated log evidence is from app stdout via `docker compose logs app`. This matches the root README limitation.

Tail-sampling math:

For `N` traces/sec, the collector keeps:

```text
N * (P(error) * 1.0 + P(slow and not error) * 1.0 + P(healthy) * 0.01)
```

With 1% errors, 1% slow traces, and 98% healthy traces:

```text
N * (0.01 + 0.01 + 0.98 * 0.01) = N * 0.0298
```

That is about 3% retention, while retaining 100% of the high-value error and slow traces.

---

## 4. Track 04 - Drift Detection

Generated files:

- `04-drift-detection/reports/drift-summary.json`
- `04-drift-detection/reports/drift-report.html`

Drift summary:

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

Test choice by feature:

- `prompt_length`: PSI for stable monitoring buckets, backed by KS when I need a distribution-shift significance test.
- `embedding_norm`: KS because it is continuous and should remain tightly distributed; small shape changes matter more than bucket movement.
- `response_length`: PSI for dashboard readability and alert thresholds, because operations teams can reason about bucket drift over time.
- `response_quality`: KL or KS. KL is useful because a quality distribution shift changes the information profile, while KS gives a simple non-parametric confirmation.

MMD is the better production choice for full embedding vectors because it compares multivariate distributions without compressing them into a single norm.

---

## 5. Track 05 - Cross-Day Integration

Screenshot created:

- `submission/screenshots/cross-day-dashboard.png`

The Day 19 and Day 20 stub exporters are running locally on ports 9101 and 9102, and Prometheus scrapes them through `host.docker.internal`. Verified metrics:

```text
day19_qdrant_collections = 3
day20_llamacpp_tokens_per_second ~= 19.5
```

The hardest prior-day metric would be Day 20 llama.cpp tokens/sec because the base server often does not expose Prometheus metrics by default. It needs either a patched server, a sidecar, or a log-to-metric adapter, so the contract is less standard than Qdrant's native `/metrics`.

---

## 6. The single change that mattered most

The single most useful change was making tracing deterministic for the lab by adding a `slow` request flag and wiring `make trace` to use it. Tail-sampling is valuable because it saves cost, but it also means healthy demo requests are often intentionally dropped. A lab that asks for a Jaeger screenshot should not rely on a 1% dice roll.

That change connects directly to the deck's sampling lesson: observability should preserve high-information events. By pushing one request above the slow-trace threshold, the stack keeps exactly the kind of trace an SRE would want during a real latency incident, while normal healthy traffic can still be sampled cheaply.
