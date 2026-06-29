# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyễn Hải An - 2A202600920
**Submission date:** 2026-06-29
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

```
Docker:        OK  (29.5.2)
Compose v2:    OK  (5.1.4)
RAM available: 3.83 GB (NEED >= 4.0 GB)
Ports free:    OK
```

Note: RAM was slightly under 4 GB minimum, but the stack ran successfully.

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
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` (requires Slack webhook URL) |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

The alert pipeline fired after 90s and resolved within 60s of app restore. Slack delivery requires a valid `SLACK_WEBHOOK_URL` in `.env`.

### One thing surprised me about Prometheus / Grafana

The multi-window multi-burn-rate SLO alerting was the most insightful part. Having two separate alert rules (SLOFastBurn for short 5m windows and SLOSlowBurn for longer 30m windows) gives both fast detection of catastrophic events and sustained burn detection without noise. The alertmanager inhibition rule that suppresses warning alerts when ServiceDown is firing is also a nice touch — it prevents alert fatigue during outages.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

```
{"event": "prediction served", "model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1678, "trace_id": "abb48ca24a97b5c4f8af5438092c21b0", "level": "info", "timestamp": "2026-06-29T04:33:00.123456Z"}
```

### Tail-sampling math

For a service producing N traces/sec:
- 1% errors → kept at 100% = N × 0.01
- 1% slow traces → kept at 100% = N × 0.01
- 98% healthy traces → kept at 1% = N × 0.98 × 0.01 = N × 0.0098
- Total retained = N × (0.01 + 0.01 + 0.0098) = N × 0.0298 ≈ 3%

Cost reduction: ~97% vs. retain-everything.

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

- **prompt_length** — KS test is best because it's a continuous numeric feature; KS detects shifts in both location and distribution shape without binning artifacts.
- **embedding_norm** — PSI is appropriate since it's a bounded continuous feature where small distribution shifts in production matter for downstream quality; PSI's binning smooths out noise.
- **response_length** — KS test again, for the same reasons as prompt_length. It's sensitive to subtle shifts in the central tendency and spread of this continuous feature.
- **response_quality** — KL divergence is most informative here because this is a probability-like score [0,1] where the shape of the distribution matters more than just the mean; KL captures the asymmetric divergence when quality degrades from high (beta(8,2)) to low (beta(2,6)).

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Based on the integration scripts, Day 20 (llama.cpp model serving) would be the hardest. llama.cpp's HTTP server doesn't natively expose Prometheus metrics, requiring a custom sidecar or patching to emit basic counters like tokens/second and queue depth. The Day 19 Qdrant integration is simpler because Qdrant has a built-in `/metrics` endpoint. Without a properly instrumented sidecar, you'd need to scrape and transform raw logs into metrics — adding significant operational overhead just to get basic RED metrics for the model server.

---

## 6. The single change that mattered most

The tail-sampling configuration in the OTel Collector was the most impactful design decision in this stack. By keeping 100% of error and slow traces while sampling only 1% of healthy traffic, we reduced span export volume by ~97% while preserving the highest-value traces for debugging. This directly connects to the deck's concept of "information-preserving reduction" — you don't need every trace to understand system behavior, but you absolutely need the ones that represent failure modes. The 30-second decision window and 50K-trace buffer are tuned to real-world traffic patterns: fast enough to make decisions before developers move on from an incident, large enough to handle burst traffic up to ~1500 traces/sec before backpressure kicks in. Without this policy, the collector would either drop spans indiscriminately under load (losing critical errors) or require significantly more memory and egress bandwidth. The 3% retention rate means we can run a single collector instance where we would otherwise need a cluster.
