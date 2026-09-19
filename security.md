---
title: Security and threat model
description: Protected assets, threat actors, and the controls that hold.
---

# Security and threat model

kubeseal-ui is a security-sensitive product: its whole purpose is keeping secret plaintext confined while
letting authorized teams edit it. This page documents the protected assets, the threat actors, and the controls
that hold.

## Protected assets

- Controller private keys
- Secret plaintext
- OIDC sessions and refresh tokens
- Typed Git credentials
- Proposal-adapter credentials
- Authorization and Git mappings
- Git history integrity
- Security-event integrity

## Threat actors

- Unauthenticated client
- Authenticated user exceeding namespace capabilities
- Compromised browser session
- Compromised API pod
- Malicious Git contributor
- Compromised repository credential
- Compromised OIDC provider or group mapping
- Compromised certificate endpoint
- Out-of-band cluster administrator

## Required controls

- Decrypt mode requires an explicit chart true/false choice; private-key RBAC renders only when enabled.
- Every reveal and mutation checks capability and Git/live equality **before** key access, cryptography, or
  any side effect.
- Per-key responses prevent unrelated plaintext from reaching the browser: reveal returns exactly one
  requested value, never the complete decrypted object.
- Plaintext, private keys, tokens, cookies, ciphertext, and diff bodies are redacted at the source; request
  bodies, response bodies, query strings, and headers are never logged.
- Runtime is non-root, read-only root filesystem, drops capabilities, uses RuntimeDefault seccomp, and runs
  with a NetworkPolicy whose egress destinations are operator-supplied.
- Git credentials are typed Secret references read from mounted files per call; tokens never appear in
  configuration, logs, or responses.
- Direct push never force-pushes; conflicts return `409` immediately with no reset, rebase, or automatic retry.
- Proposal retries use idempotency reconciliation on the deterministic branch name.
- Security events include workflow correlation (operation, actor, resource scope, result, request ID) without
  values.

## Plaintext boundary

- A revealed value reaches the browser only for the requested key, is non-cacheable
  (`Cache-Control: no-store`), and is cleared on patch, cancellation, timeout, logout, or navigation.
- Unrequested values never enter the browser.
- The server decrypts the complete Secret internally only to preserve unrelated values during a patch; the
  response never serializes it.
- Plaintext never appears in diffs, logs, errors, or security events.

## Canonical drift comparison

Editing an existing secret requires the mapped Git file to equal the live resource. The comparison ignores
`status`, `resourceVersion`, `uid`, `generation`, managed fields, timestamps, and other volatile metadata, plus
serialization order and formatting. It compares the API identity, namespace, and name, the Secret type, stable
template metadata, and `encryptedData` keys and ciphertext equality. A mismatch returns `409` with changed
field categories and redacted details; ciphertext values are omitted.

## Conditional controller-key access

With `api.enableDecrypt=true`, the API gets a Role in the controller namespace allowing `get`/`list` on
Secrets. Kubernetes RBAC cannot filter that by label, so the API can read every Secret in the controller
namespace — an accepted, visible risk only in decrypt-enabled mode. Application label checks do not reduce the
Kubernetes authorization grant. Without decrypt, no Secret permission is rendered at all.

Kyverno or Gatekeeper can further restrict Secret reads to labeled Secrets only; the chart leaves this to the
operator because it is cluster-specific.

## Disclosure

Report vulnerabilities **privately** through the affected repository's Security tab (Report a vulnerability).
Do not describe an exploitable issue in a public issue. The issue templates redirect security-sensitive
behavior to the private disclosure process and demand a redaction confirmation.

## Compromised API pod response

1. Disable decrypt mode through GitOps.
2. Revoke active sessions and rotate application session keys (`SESSION_SIGNING_KEY`).
3. Rotate Git credentials and proposal-adapter credentials, and the OIDC client secret.
4. Assess controller-key compromise and rotate according to Sealed Secrets procedures.
5. Review reveal and patch security events by user, resource, key name, and request ID.
6. Redeploy from verified images and restore only necessary capabilities.

See [runbook.md](runbook.md) for the full operational procedures.
