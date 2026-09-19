---
title: GitOps delivery
description: Namespace mappings, typed credentials, direct and proposal delivery, and ArgoCD reconciliation.
---

# GitOps delivery

kubeseal-ui delivers sealed secrets to Git through a namespace mapping that is the only server-side source of
truth: clients never choose the repository, branch, path, or delivery mode. The transport is platform-agnostic
(go-git), so any compatible remote works for fetch, branch, commit, and push.

## Namespace mapping

One mapping per namespace, in chart values:

```yaml
api:
  gitops:
    enabled: true
    authorName: kubeseal-ui
    authorEmail: kubeseal-ui@example.com
    worktreeDir: /tmp/kubeseal-ui/gitops
    namespaces:
      - namespace: payments
        repository: org/platform-repo
        branch: main
        pathTemplate: clusters/{namespace}/{name}.yaml
        authRef: platform-repo
        mode: direct
      - namespace: staging
        repository: org/platform-repo
        branch: main
        pathTemplate: clusters/{namespace}/{name}.yaml
        authRef: platform-repo
        mode: proposal
        proposalAdapter: github-pr
```

- `pathTemplate` derives the file path from the validated namespace and name; `{namespace}` and `{name}` are
  the only placeholders, and traversal (`../`) fails validation.
- `mode` is fixed by policy. Direct namespaces require `gitops:push`; proposal namespaces require
  `gitops:propose` plus a configured adapter. Users cannot downgrade a proposal namespace to direct push.
- `authRef` selects the typed credential from the list below.

The chart fails the render when gitops is enabled without namespace mappings, a credential for every mapped
`authRef`, a proposal adapter for every proposal namespace, or the credential Secret.

## Typed credentials

```yaml
api:
  gitops:
    credentials:
      - authRef: platform-repo
        mode: https-token        # https-token | ssh-agent | none
        username: kubeseal-ui    # Basic username (https-token); optional
        tokenFile: /var/run/secrets/git/platform/token
    credentialSecretName: kubeseal-ui-git
```

Credentials are stored in Kubernetes Secrets, never in values or the rendered manifest. The chart mounts each
token file's directory read-only. The token file is read on every call, so rotating the Secret takes effect
without a restart.

- `https-token`: the configured username with the token as the HTTP Basic password, which supports hosts whose
  token username convention differs.
- `ssh-agent`: the agent socket must be reachable from the container.
- `none`: only valid for remotes that accept anonymous pushes.

An unknown `authRef` is an error, never an implicit anonymous push.

## Create a Kubernetes Secret for the credentials

```bash
kubectl -n kubeseal-ui create secret generic kubeseal-ui-git \
  --from-file=platform/token=./platform-token
```

The file name must match the last path component of `tokenFile`.

## Direct delivery

A direct namespace requires `gitops:push`.

1. The API fetches the mapped base branch and records the base commit.
2. The client submits the change with the base commit it edited against.
3. A stale base returns `409` immediately — no force, reset, rebase, or automatic retry.
4. The change is committed with the configured author identity and pushed without force.
5. The remote branch is read back and the commit verified.
6. The response carries `mode`, `commit_sha`, `branch`, `file_path`, and `argocd_sync_verified: false`.

## Proposal delivery

A proposal namespace requires `gitops:propose` and a configured adapter
([proposal-adapters.md](proposal-adapters.md)).

1. The change is pushed to a server-derived branch, `kubeseal-ui/<namespace>-<name>` — never the mapped direct
   branch.
2. The adapter opens the host review object (a GitHub pull request for `type: github`).
3. The response carries the branch, commit, and `proposal_url`.
4. A host failure after the push returns `502 PROPOSAL_FAILED` with the branch left in place; retrying with a
   fresh `Idempotency-Key` reconciles onto the same branch instead of duplicating it.

## Dry run

`POST /api/v1/gitops/dry-run` returns the encrypted diff, resolved path, base commit, and fixed delivery mode
without any remote-side effect. Plaintext never appears in the diff.

## ArgoCD

A verified commit or a merged proposal does not mean ArgoCD synchronized. The delivery response always carries
`argocd_sync_verified: false` because the API never talks to ArgoCD. Check repository refresh, Application
health, SealedSecret events, and controller logs separately — ArgoCD and SealedSecret health are two
independent checks. For proposals, merge first.

## Security events

Every direct delivery and proposal attempt emits a structured stdout security event with operation, user,
namespace, secret, repository identifier, delivery mode, result, request ID, and commit or proposal identifier
when available. Tokens, plaintext, ciphertext, and diff contents are excluded.
