---
title: Runbook
description: Operational failure modes with the check that decides each one.
---

Operational failure modes with the check that decides each one. The API logs one structured JSON line per
request with a `request_id`; quote it when asking for help. Security events carry identity and resource fields
only, never values.

## Health semantics

- `/healthz` — liveness: the process is up. Always unauthenticated.
- `/readyz` — readiness: OIDC issuer + client id are configured. Until then, kubelet does not route traffic.
- `/metrics` — Prometheus exposition on the main port; 503 when observability is not configured. See
  [observability.md](observability.md).

Check ArgoCD and SealedSecret health as two independent checks, separate from API health: a verified commit or
merged proposal does not mean ArgoCD synchronized.

## Access failure

1. Locate the request ID in stdout logs.
2. Confirm exact OIDC group membership (`OIDC_GROUPS_CLAIM` must contain the groups your policy maps).
3. Confirm namespace mapping and role capabilities.
4. Confirm the required capability matches the namespace delivery mode (`gitops:push` for direct,
   `gitops:propose` for proposal).
5. Fix configuration in Git and wait for the change to take effect.

## Git/live drift

Editing an existing secret requires exactly one mapped Git file whose canonical SealedSecret matches the live
resource. The error code tells you which side is wrong:

- `GIT_SOURCE_MISSING`: no file at the derived path on the mapped branch.
- `GIT_LIVE_DRIFT`: the Git and live content differ. Reconcile first, then edit.
- identity mismatch: the `namespace`/`name` inside the manifest does not match the route; fix the manifest.
- pending ArgoCD reconciliation: the commit landed but the controller has not applied it yet.
- out-of-band cluster mutation: someone edited the live SealedSecret; restore one side first.

1. Read the mapped repository, branch, and derived path from diagnostics.
2. Compare the mapped Git manifest with the live SealedSecret.
3. Check ArgoCD sync and health.
4. Identify out-of-band cluster changes or an incorrect mapping.
5. Reconcile through Git. Do not bypass drift protection.

## Reveal or patch failure

1. Confirm `ENABLE_DECRYPT=true` ("true" is the only value that enables it).
2. Confirm the `secret:decrypt` capability; patch also requires `secret:seal`, because the server decrypts the
   complete Secret internally to preserve unrelated values.
3. Verify conditional controller-key RBAC and active-key selection (`KUBESEAL_ACTIVE_KEY_LABEL` must match a
   key Secret in `KUBESEAL_CONTROLLER_NAMESPACE`).
4. Inspect the security event by request ID without exposing values.
5. Treat historical-key failure as unsupported rather than corrupt ciphertext.

Reveal returns exactly one requested value, never the whole decrypted object. Every attempt emits a security
event with operation, actor, scope, result, and request ID; a missing event means the request never reached the
handler (check auth and CSRF middleware first).

## Direct delivery failure

1. Verify repository mapping and typed authentication Secret (`GITOPS_CREDENTIAL_REFS` shape, mounted token
   file, non-empty).
2. For HTTPS tokens, verify the configurable username and token-as-password convention.
3. Check stale-base or non-fast-forward `409` — conflicts return immediately with no force, reset, rebase, or
   automatic retry.
4. Reconcile unknown outcomes by reading the remote branch.
5. Verify ArgoCD separately after success.

## Proposal delivery failure

1. Confirm namespace mode is `proposal` and the user has `gitops:propose`.
2. Verify the generic branch push succeeded — the branch name is `kubeseal-ui/<namespace-path>`.
3. Verify the configured proposal adapter and its credential:
   - `read github token` means the mounted token file is missing; `github token file is empty` means it is
     blank.
   - A GitHub fine-grained PAT needs Contents read/write and Metadata read.
   - GitHub Enterprise Server needs the adapter `baseUrl` (`https://<host>/api/v3`); otherwise calls go to
     `api.github.com`.
   - The repository identifier must be exactly `owner/repo`; anything else fails before the host is called.
4. If a branch exists without a proposal, retry with a fresh idempotency key; the deterministic branch name
   reconciles onto the same branch instead of creating another one.

## NetworkPolicy blocks traffic

The chart refuses to guess egress destinations. `networkPolicy.enabled` requires
`networkPolicy.egress.cidrs` or `networkPolicy.egress.namespaceSelectors` (Kubernetes API, OIDC issuer, Git
remotes, proposal APIs). DNS egress to `networkPolicy.dnsNamespace` (default `kube-system`) is always rendered;
removing it breaks every service lookup. An empty `api.ingress`/`ui.ingress` admits every pod, so narrow it for
production.

## Security incident involving API private-key access

1. Disable decrypt mode through GitOps.
2. Revoke active sessions and rotate application session keys (`SESSION_SIGNING_KEY`).
3. Rotate Git credentials and proposal-adapter credentials, and the OIDC client secret.
4. Assess controller-key compromise and rotate according to Sealed Secrets procedures.
5. Review reveal and patch security events by user, resource, key name, and request ID.
6. Redeploy from verified images and restore only necessary capabilities.

## Support boundary

Community support only: GitHub issues, no hosted service, no availability SLO. Operators own log retention,
backups, Git host availability, OIDC availability, ArgoCD operation, and Kubernetes recovery.
