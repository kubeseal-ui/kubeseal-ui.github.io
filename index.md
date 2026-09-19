---
title: kubeseal-ui documentation
description: Team-oriented administration interface for Git-managed Kubernetes SealedSecrets.
---

Public documentation for kubeseal-ui, a team-oriented administration interface for Git-managed Kubernetes
SealedSecrets. Operators install the chart, wire OIDC, and map namespaces to Git repositories; team members
then update individual secret values without seeing unrelated plaintext.

## Repositories

| Repository | Contents |
|------------|----------|
| [`kubeseal-ui/api`](https://github.com/kubeseal-ui/api) | Go backend: OIDC sessions, capability checks, sealing, Git delivery, observability |
| [`kubeseal-ui/charts`](https://github.com/kubeseal-ui/charts) | Helm chart: API + UI deployments, conditional RBAC, NetworkPolicy, ServiceMonitor |
| [`kubeseal-ui/frontend`](https://github.com/kubeseal-ui/frontend) | Vue 3 SPA served by nginx |

## Getting started

1. **Install the chart** — [install.md](install.md) covers the minimal install, the values that fail the
   render, decrypt mode, and image pinning.
2. **Wire OIDC** — [oidc.md](oidc.md) covers issuer registration, the callback URI, sessions, and CSRF.
3. **Map namespaces to Git** — [gitops.md](gitops.md) covers the namespace mapping, typed credentials, direct
   and proposal delivery.
4. **Add a proposal adapter** — [proposal-adapters.md](proposal-adapters.md) covers the GitHub adapter, PAT
   scopes, and fail-closed rules.
5. **Operate** — [runbook.md](runbook.md) covers health, drift, reveal and delivery failures, ArgoCD
   reconciliation, and the private-key-compromise response.
6. **Monitor** — [observability.md](observability.md) covers the metrics, traces, logs, ServiceMonitor, and
   alerts.

The sidebar carries the full page list. Community support runs through GitHub issues on the affected
repository: no hosted service, no availability SLO. Report vulnerabilities privately through the repository's
Security tab. The issue templates demand version information, sanitized mappings, request IDs, and redacted
logs; never attach plaintext, ciphertext, tokens, cookies, or credential-bearing URLs.
