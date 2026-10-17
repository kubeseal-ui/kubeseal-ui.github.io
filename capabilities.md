---
title: Capabilities and roles
description: What each capability allows, built-in roles, and the platform-admin boundary.
---

kubeseal-ui separates plaintext, platform, and Git-delivery permissions. Roles are convenience bundles only;
the API checks capabilities, never role names, so a custom role must list every capability it needs.

## Capabilities

| Capability | Allows |
|------------|--------|
| `metadata:read` | list and inspect encrypted namespaces and SealedSecrets |
| `secret:seal` | create new sealed secrets, and add, replace, or delete keys in an existing one |
| `secret:decrypt` | reveal exactly one existing value (requires `ENABLE_DECRYPT=true`) |
| `gitops:propose` | deliver through a proposal adapter (pull/merge request) |
| `gitops:push` | deliver directly to the mapped branch |
| `access:manage` | platform diagnostics, and cluster-wide creation or editing with `secret:seal` |

Capability resolution starts from the OIDC groups claim. How a group becomes capabilities depends on whether
the deployment mounts a policy document:

- **With `CONFIG_PATH` set** the document maps group names to roles — built-in or custom — and a rule may
  scope its grant to particular namespaces, or to any namespace. The API refuses to boot on a document it
  cannot parse rather than serving one the operator did not write, and re-reads it on `SIGHUP`. Mounting a
  document is a cutover, not an addition: while one is loaded, a group name that merely matches a built-in
  role grants nothing.
- **With no document**, a group grants a bundle only when it is named exactly after a built-in role. An
  unmatched group silently yields no capabilities.

Matching groups contribute a union of capabilities.

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
- Patching a key requires `secret:seal` only. The server never decrypts: `add` and `replace` seal just the
  value they introduce, `delete` drops one entry, and every other entry is carried through byte-for-byte.
  `secret:decrypt` is reveal-only.
- Release duties split further: `gitops:propose` (branch + review object) and `gitops:push` (direct delivery)
  are independent capabilities, so a team can require review before delivery.

## Editing workflow (what the user sees)

1. Open a Git-managed SealedSecret whose Git and live states match. The mode switch above the list flips the
   document between **Key rows** and the read-only **YAML** the cluster holds.
2. Key names are listed; values remain concealed.
3. Reveal a key only when needed. The browser receives only that key, and **Conceal** drops it again. A reveal
   is a look, not a change — it stages nothing.
4. Stage each change inside its own key row: **Change** an existing key (Replace or Delete), **Add key** for a
   key the Secret does not have yet, **Remove** to stage a deletion directly, and **Discard** or **Discard
   all** to undo. A value longer than one line — a private key, a kubeconfig, a config file — needs the
   **lines** control beside the value field, which opens it into a box several lines tall. The one-line field
   is a password field, and HTML drops newlines from those, so a block pasted into it would arrive silently
   joined into a single line.
5. Press the button at the foot of the page once for everything staged together. It carries the name of the
   next press rather than the name of a stage, so there is nothing to work out about where you are:
   **Review change** sends the batch as a single request and names the keys and operations it covers, never
   values; **Apply & check** confirms the encrypted patch — the server edits only the entries you staged and
   leaves the rest of the manifest untouched —
   and runs the dry run against the branch in the same press; **Deliver to Git**, or **Create proposal** where
   the namespace delivers by proposal, delivers through the namespace policy. **Edit**, on the workflow rail
   above and in the bar over the review or the check, drops the review and the check and returns you to the
   rows with everything staged still there; the button at the foot never steps back.
6. Review the encrypted Git diff beside the document. The panel names the file the check resolved and the
   branch, so the destination is on screen before the delivery press.
7. Check ArgoCD and SealedSecret reconciliation separately.

Creating a Secret runs the same review-and-deliver flow, from a manifest rather than a key list. The **Name**
field and the manifest's `metadata.name` are one value in two places, and whichever is written last holds:
typing in the field moves the line, and writing or pasting the line moves the field. They cannot disagree,
which matters because the server refuses a manifest whose name is not the one the request carries. Once
encrypted, only ciphertext is held: the form folds to a summary row with **Edit** to reopen the manifest, and
**Discard encrypted draft** clears the name and target directory with it.

Every reveal, patch, and delivery attempt creates a security event without recording values. Plaintext is
non-cacheable and never stored in browser persistence; a revealed value is cleared on patch, cancellation,
timeout, logout, or navigation. Unrequested values never enter the browser.
