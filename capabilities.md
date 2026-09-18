# Capabilities and roles

kubeseal-ui separates plaintext, platform, and Git-delivery permissions. Roles are convenience bundles only;
the API checks capabilities, never role names, so a custom role must list every capability it needs.

## Capabilities

| Capability | Allows |
|------------|--------|
| `metadata:read` | list and inspect encrypted namespaces and SealedSecrets |
| `secret:seal` | create new sealed secrets at vacant mapped paths |
| `secret:decrypt` | reveal exactly one existing value (requires `ENABLE_DECRYPT=true`) |
| `secret:seal` + `secret:decrypt` | patch one key and preserve the rest |
| `gitops:propose` | deliver through a proposal adapter (pull/merge request) |
| `gitops:push` | deliver directly to the mapped branch |
| `access:manage` | platform diagnostics and cluster-wide sealing with `secret:seal` |

Capability resolution comes from the OIDC groups claim: each group maps to a role, and the role lists
capabilities. Matching groups contribute a union of capabilities.

## Built-in roles

| Role | Capabilities |
|------|--------------|
| viewer | `metadata:read` |
| editor | `metadata:read`, `secret:seal` |
| secret-manager | `metadata:read`, `secret:seal`, `secret:decrypt` |
| release-proposer | `metadata:read`, `gitops:propose` |
| release-pusher | `metadata:read`, `gitops:push` |
| platform-admin | `metadata:read`, `secret:seal`, `access:manage` |

## Platform-admin boundary

Administrative authority does not implicitly grant access to every namespace's sensitive payload:

- platform-admin does **not** implicitly decrypt all secrets — `secret:decrypt` is a separate capability that
  must be granted explicitly
- platform-admin does **not** implicitly push to Git — `gitops:push`/`gitops:propose` are separate
- platform-admin gains cluster-wide **sealing** (with `secret:seal`) and diagnostics, not team plaintext

This is pinned by tests: administrators lack implicit sensitive access, and the UI renders capability-gated
actions only for users who hold the capability.

## Namespace boundaries

- A capability is necessary but not sufficient: delivery mode is fixed by the namespace policy, and the
  namespace's Git mapping must exist and match the live resource.
- Direct namespaces cannot propose; proposal namespaces cannot push directly.
- A namespace policy may fix which mode applies; users cannot downgrade a proposal namespace to direct push.

## Custom roles

Custom roles are complete capability lists for teams that need separation of duties — for example, a role that
can prepare secrets (`secret:seal`) but neither decrypt nor deliver. Group rules map exact identity-provider
group names to one role; duplicate role definitions and unknown role references fail validation.

## Separation of duties

- Content preparation (`secret:seal`) is separate from sensitive read (`secret:decrypt`) and release
  (`gitops:push` / `gitops:propose`).
- Patching one key requires both `secret:seal` and `secret:decrypt`, because the server decrypts the complete
  Secret internally to preserve unrelated values.
- Release duties split further: `gitops:propose` (branch + review object) and `gitops:push` (direct delivery)
  are independent capabilities, so a team can require review before delivery.

## Editing workflow (what the user sees)

1. Open a Git-managed SealedSecret whose Git and live states match.
2. View key names; values remain concealed.
3. Reveal a key only when needed. The browser receives only that key.
4. Enter the new value for the selected key.
5. Patch and reseal. The server preserves unrelated values internally.
6. Review the encrypted Git diff.
7. Deliver using the namespace policy: direct returns a verified commit; proposal returns a review URL.
8. Check ArgoCD and SealedSecret reconciliation separately.

Every reveal, patch, and delivery attempt creates a security event without recording values. Plaintext is
non-cacheable and never stored in browser persistence; a revealed value is cleared on patch, cancellation,
timeout, logout, or navigation. Unrequested values never enter the browser.
