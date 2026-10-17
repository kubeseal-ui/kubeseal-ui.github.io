---
title: Capability matrix
description: Every API operation and built-in role against the six capabilities, and the scope rules that decide where a grant applies.
---

This is the reference form of kubeseal-ui's authorization model: which capability each operation requires,
which built-in roles carry which capabilities, and where a grant applies. The narrative version, with the
platform-admin boundary and the editing workflow, is [Capabilities and roles](capabilities.md).

## The six capabilities

| Capability | Allows | Gate |
|------------|--------|------|
| `metadata:read` | list namespaces and SealedSecrets, open one, read Git/live status | — |
| `secret:seal` | create a SealedSecret; add, replace, or delete keys in an existing one | public certificate only; the private key is never loaded |
| `secret:decrypt` | reveal exactly one existing value | `ENABLE_DECRYPT=true` |
| `gitops:push` | dry-run, deliver, and write live state into Git for a **direct** namespace | namespace mode is `direct` |
| `gitops:propose` | the same for a **proposal** namespace | namespace mode is `proposal` |
| `access:manage` | platform-sensitive actions; in this release, the cluster-wide creation and editing check | must be a global grant (see [Scopes](#scopes-where-a-grant-applies)) |

## Operation matrix

`✓` marks a capability the request is checked against, in the namespace it acts on. A dash or empty cell
is not consulted.

| Operation | Endpoint | metadata:read | secret:seal | secret:decrypt | gitops:push | gitops:propose | access:manage |
|-----------|----------|:---:|:---:|:---:|:---:|:---:|:---:|
| Browse namespaces | `GET /api/v1/namespaces` | ✓ | | | | | |
| List secrets | `GET /api/v1/secrets` | ✓ | | | | | |
| Open a sealed secret | `GET /api/v1/secrets/{namespace}/{name}` | ✓ | | | | | |
| List Git target paths | `GET /api/v1/gitops/paths` | ✓ | | | | | |
| Git/live status | `GET /api/v1/gitops/sync` | ✓ | | | | | |
| Preview a staged edit | `POST /api/v1/secrets/{namespace}/{name}/diff` | | ✓ | | | | ✓ cluster-wide |
| Apply a staged edit | `PATCH /api/v1/secrets/{namespace}/{name}/values` | | ✓ | | | | ✓ cluster-wide |
| Reveal one value | `POST /api/v1/secrets/{namespace}/{name}/reveal` | | | ✓ | | | |
| Create a sealed secret | `POST /api/v1/secrets/encrypt` | | ✓ | | | | ✓ cluster-wide |
| Dry-run a delivery | `POST /api/v1/gitops/dry-run` | | | | ✓ direct | ✓ proposal | |
| Deliver to Git | `POST /api/v1/gitops/deliver` | | | | ✓ direct | ✓ proposal | |
| Write live state into Git | `POST /api/v1/gitops/sync` | | | | ✓ direct | ✓ proposal | |

Notes on the matrix:

- **The two delivery columns are alternatives, fixed by the namespace mapping.** A `direct` mapping checks
  `gitops:push`; a `proposal` mapping checks `gitops:propose`. The other is never consulted, and the caller
  cannot choose the mode.
- **`access:manage` is consulted only for a cluster-wide SealedSecret**, alongside `secret:seal`. It is
  checked against the global grant set, not the namespace's — cluster-wide values belong to no namespace.
- **Reveal also needs the deployment flag.** `ENABLE_DECRYPT=false` refuses a permitted reveal with
  `403 DECRYPT_DISABLED` even when the capability is held.
- **The browse and list rows are pre-filters.** Holding `metadata:read` somewhere admits the request; each
  returned namespace or secret is then filtered to the ones you hold it in, so the endpoint never returns
  more than you can read.
- **A capability is necessary, not sufficient.** Editing additionally requires Git and live state to match;
  creating requires the derived Git path to be vacant. Those are resource preconditions, not capabilities,
  and no grant removes them.

## Role matrix

| Role | metadata:read | secret:seal | secret:decrypt | gitops:push | gitops:propose | access:manage |
|------|:---:|:---:|:---:|:---:|:---:|:---:|
| `viewer` | ✓ | | | | | |
| `editor` | ✓ | ✓ | | | | |
| `secret-manager` | ✓ | ✓ | ✓ | | | |
| `release-proposer` | ✓ | | | | ✓ | |
| `release-pusher` | ✓ | | | ✓ | | |
| `platform-admin` | ✓ | ✓ | | | | ✓ |

What separates them:

- **viewer** sees that a SealedSecret exists and which keys it holds, never a value.
- **editor** stages add, replace, and delete. Editing seals only the value being introduced and carries every
  other entry through untouched, so it needs no private key and never reads an existing value. `secret:seal`
  alone is enough for any edit.
- **secret-manager** adds reveal to that. Reveal is a look, not a change.
- **release-proposer** and **release-pusher** deliver sealed manifests and stage nothing, so they hold
  neither seal nor decrypt — pushing ciphertext is not reading plaintext.
- **platform-admin** carries sealing and platform-sensitive actions. Decrypt and delivery stay separate
  capabilities it does not imply.

## Scopes: where a grant applies

Every grant is namespace-scoped. The effective set in namespace `N` is the **global** set unioned with the
grants **scoped** to `N`:

- **Global** grants apply everywhere. They come from a rule with `namespaces: ['*']`, or — when no policy
  document is loaded — from an OIDC group named exactly after a built-in role.
- **Scoped** grants apply only in the namespaces their rule names, as exact strings. Prefix globs and group
  hierarchies are not matched.
- A namespace absent from the scoped set is not denied: the global set still applies there.
- An unnamed or unknown namespace is not a wildcard. Checking a capability "in no namespace" answers from
  the global set alone.

That last point is what the cluster-wide rule uses. A cluster-wide SealedSecret is readable in every
namespace, so editing one requires `access:manage` in the **global** set — a rule scoped to `immich` does not
supply it, even though it grants `secret:seal` there.

Delivery is checked in the mapped namespace, not globally: a `gitops:push` grant in `development` does not
deliver from `payments`.

## Custom roles

Custom roles are complete capability lists with no inheritance, and a built-in role name may be listed only
with its canonical bundle. A team that needs editing plus proposal delivery, scoped to two namespaces:

```yaml
authz:
  roles:
    immich-developer:
      capabilities: [metadata:read, secret:seal, gitops:propose]

  groups:
    - name: immich-developer
      namespaces: [immich, tools]
      role: immich-developer
```

That group can add, replace, and delete keys in `immich` and `tools`, propose delivery where those namespaces
are mapped for proposal, and do nothing else — no reveal, no direct push.

To let it also edit a cluster-wide SealedSecret, grant `access:manage` globally through a **separate** role.
Reusing the team role in a `['*']` rule would grant its seal and propose capabilities everywhere as well:

```yaml
authz:
  roles:
    platform-access:
      capabilities: [access:manage]

  groups:
    - name: immich-developer
      namespaces: ['*']
      role: platform-access
```

## Checking what you hold

`GET /api/v1/auth/me` returns both halves of the grant:

- `capabilities` — the global set;
- `namespaces` — the per-namespace map, always present and empty when nothing is scoped.

The effective grant in a namespace is the union, and the UI applies the same union before rendering an
action. A refused action names the missing capability in its `403 CAPABILITY_DENIED` message;
[Troubleshooting](troubleshooting.md) covers what each refusal means.
