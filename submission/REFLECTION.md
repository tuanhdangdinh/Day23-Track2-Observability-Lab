# Day 23 Lab Reflection

**Student:** Tuan H. Dang Dinh
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/tuanhdangdinh/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.1)
Compose v2:    OK  (5.1.3)
RAM available: 7.65 GB (OK)
Ports free:    OK
Report written: 00-setup/setup-report.json
```

setup-report.json summary:
```json
{
  "docker": {"ok": true, "version": "29.4.1"},
  "compose_v2": {"ok": true, "version": "5.1.3"},
  "ram_gb_available": 7.65,
  "ram_ok": true,
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

The AI Service Overview dashboard renders 6 panels: Request Rate (RPS), Latency P50/P95/P99, Error Rate %, Active Gauge, Token Throughput, and GPU Utilization. After `make load` (1065 requests over 60s), all panels showed non-zero data.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

With zero errors during normal load, the burn-rate panels all read near 0. The recording rules `inference:fail_ratio:rate5m` and friends are correctly computed from the counter ratio.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` via `docker stop` | trigger-alert.sh step 1 |
| T0+80s | `ServiceDown` fired in Alertmanager | alert fired after 16×5s |
| T1 | restored app via `docker start` | trigger-alert.sh step 3 |
| T1+5s | alert resolved | trigger-alert.sh confirmed |

### One thing surprised me about Prometheus / Grafana

Grafana 11.x returns `"database": "ok"` (with a space) in the health endpoint JSON, but the original smoke-check grepped for `"database":"ok"` (no space), causing a false failure. This is a reminder that even health endpoints have subtle format contracts — parsing JSON properly instead of grepping strings is always safer.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans under the parent `predict` span.

The trace from `make trace` has trace_id: `df9ef54bd7630be5dc155d9b8722805a`.

### Log line correlated to trace

```json
{"model": "llama3-mock", "input_tokens": 8, "output_tokens": 8, "quality": 0.855, "duration_seconds": 0.171, "trace_id": "73a324b5c493b836ab80bf422301f3e6", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T12:56:53.813491Z"}
```

The `trace_id` field in every structured log line ties directly to the corresponding Jaeger trace. In Grafana, the Loki datasource is configured with a derived field regex `"trace_id":"([a-fA-F0-9]+)"` that turns the trace_id into a clickable link to Jaeger.

### Tail-sampling math

During `make load` the app served ~17.7 req/s (1064 requests / 60s). The OTel Collector tail-sampling policy keeps:
- All ERROR status traces (forced failures)
- All traces with latency > 2000ms (none in this run — max was ~310ms)
- 1% of remaining healthy traces probabilistically

So at 17.7 req/s with no errors or slow traces: **expected retention ≈ 0.177 traces/sec**, or about 11 traces over the 60s load test. The other ~1053 were sampled out. This is exactly the point: at production scale (1000+ RPS) you cannot store every trace; keeping 1% of healthy + 100% of anomalies gives you both statistical coverage and complete signal on problems.

---

## 4. Track 04 — Drift Detection

### PSI scores

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

| Feature | Best test | Reason |
|---|---|---|
| `prompt_length` | **KS** | Continuous, unbounded numeric. KS is distribution-free and detects shifts in the shape of a continuous CDF without assuming normality. PSI also works well here (PSI=3.46 >> 0.2 threshold). |
| `embedding_norm` | **KL divergence** | Near-Gaussian continuous. KL quantifies how much information is "lost" when approximating the reference distribution with the current one — ideal for well-behaved numeric features. PSI=0.019 correctly shows no drift. |
| `response_length` | **PSI** | Numeric, bounded in practice. PSI was designed for credit-scoring feature monitoring (also continuous numeric). PSI=0.016 correctly shows no drift. |
| `response_quality` | **MMD (Maximum Mean Discrepancy)** | A score in [0,1] with a Beta distribution shape. KS is valid but MMD is more powerful for comparing distributions in kernel-feature space, especially when the shift is in the shape parameter (beta(8,2) vs beta(2,6)) rather than just the mean. PSI=8.85 and KS=0.941 both scream drift, which matches. |

Key rule of thumb: use **PSI** for numeric features in production monitoring (easy to interpret: <0.1 = stable, 0.1–0.2 = investigate, >0.2 = act). Use **KS** when you need a p-value for a statistical test. Use **KL** when you care about the information-theoretic divergence. Use **MMD** for high-dimensional or non-parametric distributions (embeddings, images).

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The hardest would be Day 19 (Qdrant vector store). Qdrant's `/metrics` endpoint requires the service to be running on a known host:port, and in a Docker network environment, `host.docker.internal` resolution differs between Mac and Linux. The stub in `monitor-day19-vector-store.py` generates synthetic Qdrant-style metrics because the actual Qdrant service is not part of this lab's Compose stack. A real integration would require either adding Qdrant to `docker-compose.yml` or relying on `extra_hosts: ["host.docker.internal:host-gateway"]` — which only works on Linux Docker, not Mac Docker Desktop.

---

## 6. The single change that mattered most

The single change that made the biggest practical difference was **fixing the Alertmanager configuration to start correctly**. The original `alertmanager.yml` used `{{ env "SLACK_WEBHOOK_URL" }}` Go template syntax inline in the `api_url` field. Alertmanager's config parser treats this as a literal string, not a template — and since `{{` is not a valid URL scheme, the config fails to load and the container exits. This silently took down the entire alerting pipeline: no alertmanager meant no alert routing, no Slack notifications, and the smoke check failing on port 9093.

The fix — moving to `global.slack_api_url` with a valid placeholder URL — is conceptually simple but illustrates a deep point from the deck's §5 (SLO + Burn-Rate): **an alert system that fails to start gives you false confidence**. You think you have coverage (the Prometheus rules fire correctly, the `/alerts` endpoint shows the alert state), but the notifications never leave the building. This is the observability equivalent of a smoke detector with a dead battery: it looks fine until you need it. The lesson is that the alerting pipeline itself must be monitored end-to-end — which is why the lab wires `make smoke` and `make verify` to check all 7 services independently, not just the app.
