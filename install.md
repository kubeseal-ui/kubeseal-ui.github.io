# Installation

## Prerequisites

- Kubernetes 1.27+ (chart `kubeVersion: >=1.27.0-0`)
- A Sealed Secrets controller (0.39+ recommended; kubeseal-ui reads its active key when decrypt is enabled)
- An OIDC provider (Authentik, Keycloak, Okta, Auth0, Google, Azure — any OpenID Connect provider)
- A Git repository the API can push to (any remote supported by go-git)
- cert-manager or another TLS source for the ingress (optional)

## Minimal install

```bash
helm install kubeseal-ui oci://ghcr.io/kubeseal-ui/charts/kubeseal-ui \
  --namespace kubeseal-ui --create-namespace \
  --set api.env.OIDC_ISSUER=https://auth.example.com \
  --set api.env.OIDC_CLIENT_ID=kubeseal-ui \
  --set api.enableDecrypt=false
```

Or from a checkout of the charts repo:

```bash
helm install kubeseal-ui . \
  --namespace kubeseal-ui --create-namespace \
  --set api.env.OIDC_ISSUER=https://auth.example.com \
  --set api.env.OIDC_CLIENT_ID=kubeseal-ui \
  --set api.enableDecrypt=false
```

The chart installs one API deployment (Go backend) and one UI deployment (nginx-served SPA) with a Service
each. Both containers run non-root with read-only root filesystems, dropped capabilities, and RuntimeDefault
seccomp.

## Decrypt mode

`api.enableDecrypt` must be an explicit choice — reveal and patch require it, and the conditional
controller-key RBAC renders only when it is true.

```bash
--set api.enableDecrypt=true \
--set api.controllerNamespace=cluster
```

With decrypt enabled, the API gets a Role in the controller namespace allowing `get`/`list` on Secrets, which
is what reveal and patch need to read the controller's active key. Kubernetes RBAC cannot filter that by
label, so the API can read every Secret in the controller namespace — an accepted, visible risk only in
decrypt-enabled mode. Without decrypt, no Secret permission is rendered at all.

## Values that fail the render

The chart refuses to render a configuration it cannot serve:

- `api.gitops.enabled` with no namespace mappings or no credential Secret
- a namespace `authRef` with no matching credential entry
- `https-token` credentials with no `tokenFile`
- an unknown credential mode
- a namespace `mode` other than `direct` or `proposal`
- a `proposal` namespace with no `proposalAdapter`, or one that names an adapter absent from
  `api.gitops.proposalAdapters`
- a `direct` namespace that declares `proposalAdapter`
- a proposal adapter with an unknown `type`, a missing `tokenFile`, or a duplicate `name`
- `networkPolicy.enabled` with no `networkPolicy.egress` destination

## Image pinning

`api.image.tag` and `ui.image.tag` are empty by default, which resolves to `Chart.appVersion` — a `-dev`
marker with no published image. Pin real tags or digests for every environment:

```yaml
api:
  image:
    tag: sha-8618c2e
ui:
  image:
    tag: sha-2a736c7
```

API and UI release on independent cadences; pin them separately.

## Observability

```yaml
observability:
  serviceMonitor:
    enabled: true
    selector:
      release: prometheus
  prometheusRule:
    enabled: true
    selector:
      release: prometheus
  otlpEndpoint: http://otel-collector.observability.svc:4317
  traceSampleRatio: "0.25"
```

The API serves Prometheus text at `/metrics` on its main port; ServiceMonitor scrapes it. Traces and logs go
to the OTLP collector. Without an endpoint the SDK stays unmounted and `/metrics` returns 503. See
[observability.md](observability.md).

## NetworkPolicy

Disabled by default. When enabled, the chart requires explicit egress destinations because the Kubernetes API,
OIDC issuer, Git remotes, and proposal APIs are environment specific:

```yaml
networkPolicy:
  enabled: true
  api:
    ingress:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: ingress-nginx
  egress:
    cidrs:
      - 10.43.0.1/32
    namespaceSelectors: []
    otlp: []
```

DNS egress to `networkPolicy.dnsNamespace` (default `kube-system`) is always rendered.

## Ingress

The chart ships no ingress template yet; the UI Service is ClusterIP. Terminate TLS on your ingress
controller and route `/` to the UI Service and `/api` to the API Service, or front both with an
IngressRoute/Traefik configuration matching your cluster. The API's cookies are `Secure` and `HttpOnly`, so
the UI must be reached over HTTPS, and `CSRF_TRUSTED_ORIGINS` must contain the browser origin.

## Verify

```bash
helm lint .
helm template test-release . > /dev/null
bash scripts/validate.sh

kubectl -n kubeseal-ui get pods
curl https://kubeseal-ui.example.com/healthz     # via your ingress
curl https://kubeseal-ui.example.com/metrics    # 200 with observability, 503 without
```

## Upgrade

```bash
helm upgrade kubeseal-ui . -n kubeseal-ui -f values.yaml
```

The chart fails closed on an unservable configuration, so an upgrade with broken values leaves the previous
release running. Sessions are signed with `SESSION_SIGNING_KEY`; keep it stable across restarts or every user
is redirected to login.
