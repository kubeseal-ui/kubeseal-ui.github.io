---
title: OIDC setup
description: Register the application, configure the chart, sessions, CSRF, and group claims.
---

kubeseal-ui uses OpenID Connect with the authorization code flow and PKCE. It works with any OpenID Connect
provider; the SPA never sees a token, client id, or provider URL because the flow runs server-side.

## Register the application

Register a confidential client with your provider (Authentik, Keycloak, Okta, Auth0, Google, Azure):

- **Client type**: confidential (server-side)
- **Redirect URI**: `https://<ui-host>/api/v1/auth/callback` — must match `OIDC_REDIRECT_URL` byte for byte,
  including scheme and trailing path
- **Scopes**: `openid profile email groups` (the default; override with `OIDC_SCOPES`)
- **Grant**: authorization code + PKCE

## Configure the chart

```yaml
api:
  env:
    OIDC_ISSUER: https://auth.example.com        # the issuer URL; gates /readyz
    OIDC_CLIENT_ID: kubeseal-ui
    # Keep the client secret out of Git; use --set or a Secret.
    OIDC_CLIENT_SECRET: ""
    OIDC_REDIRECT_URL: https://kubeseal-ui.example.com/api/v1/auth/callback
    OIDC_SCOPES: "openid profile email groups"
    OIDC_GROUPS_CLAIM: groups                    # drives capability resolution
    OIDC_USERNAME_CLAIM: preferred_username      # display username
```

`OIDC_ISSUER` and `OIDC_CLIENT_ID` also gate readiness: until they are set, `/readyz` reports not ready so
kubelet does not route traffic.

## Session and CSRF

```yaml
api:
  env:
    # REQUIRED. 32+ random bytes; no /api/v1 route is mounted without it.
    SESSION_SIGNING_KEY: ""
    # Optional. Empty means host-only cookies; requests from another
    # subdomain fail.
    COOKIE_DOMAIN: ""
    # REQUIRED in practice. Space-separated origins allowed to send
    # state-changing requests. Empty leaves a placeholder origin, so the
    # real origin is rejected with 403.
    CSRF_TRUSTED_ORIGINS: "https://kubeseal-ui.example.com"
```

Generate the signing key:

```bash
openssl rand -base64 32
```

Cookies are `HttpOnly`, `Secure`, and `SameSite=Lax`; the UI must be reached over HTTPS. The CSRF token is
delivered to the SPA via `GET /api/v1/auth/csrf` and sent back as `X-CSRF-Token` on every state-changing
request.

## Group claims and capabilities

The API resolves capabilities from the OIDC groups claim through the policy store: each group maps to a role,
and the role lists the capabilities. The API checks capabilities, never role names, so a custom role must list
every capability it needs. See [capabilities.md](capabilities.md).

If a user sees "namespace or action missing", confirm their group claim actually contains the groups your
policy maps — the mapping is exact, and a missing group silently yields no capabilities.

## Verify

```bash
# Discovery works and returns the issuer you configured.
curl https://auth.example.com/.well-known/openid-configuration

# The API is ready once OIDC is configured.
curl https://kubeseal-ui.example.com/readyz

# Login redirects to the provider.
curl -sI https://kubeseal-ui.example.com/api/v1/auth/login | head -3
```

A login loop means the API and the provider disagree about the session: check discovery, the exact callback
URI, `SESSION_SIGNING_KEY` stability, `CSRF_TRUSTED_ORIGINS`, and cookie flags. See
[troubleshooting.md](troubleshooting.md).
