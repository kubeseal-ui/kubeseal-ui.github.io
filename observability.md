# Observability

kubeseal-ui emits three backend signals through **one SDK** (`go.opentelemetry.io/otel`) and **one protocol**
(OTLP gRPC): metrics, traces, and logs, all exported to a collector. Prometheus, Loki, and Tempo read from it.
The frontend reports Web Vitals separately. The wiring is optional infrastructure: with no OTLP endpoint the
SDK stays unmounted, `/metrics` returns 503, logging falls back to plain JSON with no trace attributes, and
local boots never make network calls.

## Metrics

The API serves Prometheus text at `/metrics` on its main port via the OTel Prometheus exporter. No separate
metrics port or admin route exists to misconfigure.

| Metric | Labels |
|--------|--------|
| `kubeseal_gui_http_requests_total` | `handler`, `method`, `code` |
| `kubeseal_gui_http_request_duration_seconds` | `handler`, `method` (buckets 5ms..10s) |
| `kubeseal_gui_sealed_secret_operations_total` | `operation`, `result` |
| `kubeseal_gui_gitops_delivery_total` | `mode`, `result` |
| `kubeseal_gui_openfga_check_total` | `result` |
| `kubeseal_gui_oidc_auth_total` | `result` |

Cardinality is bounded by construction: labels carry handler/method/code and bounded outcome values only.
User identities, namespaces, and secret names are excluded from metrics — they belong in the security events
and logs. `handler` is the route pattern (`/secrets/{namespace}/{name}`), never the raw path.

Enable scraping with the chart:

```yaml
observability:
  serviceMonitor:
    enabled: true
    selector:
      release: prometheus     # the Prometheus operator's selector
    interval: 30s
```

## Alerts

`observability.prometheusRule.enabled: true` renders four alerts:

| Alert | Condition |
|-------|-----------|
| `KubesealUIHighErrorRate` | 5xx rate above 5% for 10 minutes |
| `KubesealUIHighLatency` | p99 latency above 2s for 10 minutes |
| `KubesealUIGitOpsPushFailing` | more than 3 delivery failures (`failed`, `conflict`, `proposal_failed`) in 30 minutes |
| `KubesealUICryptoFailures` | more than 5 sealed-secret operation failures in 15 minutes |

## Traces

The API starts a server span per request (`SpanKindServer`) with the bounded `http.route` attribute and the
response status; 5xx marks the span as error. An inbound `traceparent` is extracted, so upstream callers
correlate into the service. Sampling is parent-based with a `TraceIDRatioBased` fallback.

```yaml
observability:
  otlpEndpoint: http://otel-collector.observability.svc:4317
  traceSampleRatio: "0.25"   # 0..1; default 0.1
```

## Logs

Every `http_request` log line carries `method`, `path`, `route`, `status`, `duration_ms`, `request_id`, and —
when a span is recording — `trace_id` and `span_id`, so every log line correlates to a trace and every metric
back to a log line. Request bodies, response bodies, query strings, and headers are never logged; the
redacting handler drops sensitive attribute keys at the source. Security events carry operation, actor,
namespace, secret, mode, result, and request ID — never values.

Logs ship to the collector through the same OTLP endpoint. Point Loki's Promtail or the OTel logs pipeline at
the collector Service.

## Configuration

| Env var (chart value) | Meaning | Default |
|-----------------------|---------|---------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` (`observability.otlpEndpoint`) | OTLP gRPC host:port; the chart strips the scheme | disabled |
| `OTEL_SERVICE_NAME` | resource `service.name` | the release's API name |
| `OTEL_SERVICE_VERSION` | resource `service.version` | image tag or appVersion |
| `OTEL_DEPLOYMENT_ENVIRONMENT` | classic `deployment.environment` | the release namespace |
| `OTEL_TRACE_SAMPLE_RATIO` (`observability.traceSampleRatio`) | parent-based sampler ratio 0..1 | 0.1 |
| `OTEL_METRIC_INTERVAL_SECONDS` (`observability.metricIntervalSeconds`) | OTLP push interval | 30 |

Invalid ratio or interval values log a warning and keep the default rather than failing the boot. Resource
attributes also carry `k8s.namespace.name` and the `telemetry.sdk.*` keys.

## NetworkPolicy

When the policy is enabled, declare the collector destination so port 4317 egress renders:

```yaml
networkPolicy:
  enabled: true
  egress:
    otlp:
      - 10.43.0.20/32        # collector ClusterIP or LB
```

DNS egress to `networkPolicy.dnsNamespace` is always rendered; the Kubernetes API, OIDC, Git, and proposal-API
destinations are operator-supplied.

## Frontend web vitals

The SPA reports CLS, INP, FCP, LCP, and TTFB as JSON batches to `VITE_TELEMETRY_ENDPOINT` (POST, `keepalive`).
The module is opt-in: unset (the default) keeps every call a no-op. Payloads carry the metric name, value,
rating, navigation type, the bounded page path, and a timestamp; no identifiers.

```yaml
# frontend build-time env
VITE_TELEMETRY_ENDPOINT: /api/v1/telemetry/vitals
```

## Verify

```bash
# 200 with observability configured; 503 without (the SDK is unmounted).
curl https://kubeseal-ui.example.com/metrics | head

# Prometheus text with the contract metrics.
curl -s https://kubeseal-ui.example.com/metrics | grep kubeseal_gui

# Log lines carry trace correlation once a span is recording.
kubectl -n kubeseal-ui logs deploy/kubeseal-ui-api | grep trace_id | head -3
```

With a live collector, check the Prometheus target is up, traces land in Tempo, and logs in Loki with
`trace_id` linking them.
