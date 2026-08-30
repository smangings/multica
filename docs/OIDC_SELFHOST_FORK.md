# OIDC Deployment Runbook (fork self-host)

Operator runbook for enabling generic OpenID Connect on our self-hosted fork.
The upstream feature reference is multica-ai/multica#5792; its official docs
(`auth-setup`, `environment-variables`) apply unchanged. This page covers only
what differs or needs sequencing on our fork (GHCR images, migrations,
rollback).

## 1. Prerequisites — images and migrations

The OIDC server code, migrations **285–287**, and the compose/Helm wiring land
together on fork `main`. Before configuring anything:

1. Pull a fork image that contains the OIDC code:
   ```bash
   # docker-compose.selfhost.yml defaults
   MULTICA_BACKEND_IMAGE=ghcr.io/benjsnellings/multica-backend
   MULTICA_WEB_IMAGE=ghcr.io/benjsnellings/multica-web
   docker compose -f docker-compose.selfhost.yml pull backend web
   ```
   Images are published by the fork's `release.yml` from `main` — do not
   attempt OIDC on an image built before the #5792 merge.
2. Migrations 285–287 run automatically on backend startup (`migrate up`).
   Verify before first OIDC login:
   ```bash
   docker compose -f docker-compose.selfhost.yml exec backend \
     psql "$DATABASE_URL" -c "\d user_oidc_identity"
   curl -fs localhost:8080/healthz   # expect "migrations":"ok"
   ```
   They are additive (`user_oidc_identity` table + two indexes); no backfill
   or downtime is required. For Helm, a single `helm upgrade` rolls the
   backend and applies them via the startup hook.

## 2. Configure the provider

Set in the self-host `.env` (compose) or `values.yaml` (Helm):

```bash
OIDC_ISSUER_URL=https://auth.yourdomain.com/application/o/multica
OIDC_CLIENT_ID=multica
OIDC_CLIENT_SECRET=replace-me
OIDC_REDIRECT_URI=https://multica.yourdomain.com/auth/callback
OIDC_PROVIDER_NAME=Company SSO
# Optional:
OIDC_ALLOWED_GROUPS=multica-users
OIDC_GROUPS_CLAIM=groups
OIDC_REQUIRE_VERIFIED_EMAIL=true
```

In the IdP, register a confidential web application whose **redirect URI
matches `OIDC_REDIRECT_URI` exactly** (scheme, host, port, path, no trailing
slash differences), grant `openid profile email`, and restart the backend.
Configuration is read at runtime via `/api/config` — no frontend rebuild.

Fork note: the fork publishes its own GHCR images but keeps the upstream
compose/Helm env plumbing verbatim, so no extra variables are needed here.

## 3. Signup allowlist behaviour

OIDC-created users go through the same signup gate as email/Google users:
`SIGNUP_MODE` (open / allowlist / closed) and `SIGNUP_EMAIL_ALLOWLIST` apply
to the verified email returned by the IdP. In allowlist mode, add the user's
IdP email before their first login or they will be rejected at account
creation. Existing accounts are linked by the stable `(issuer, subject)`
identity, so later email changes at the IdP do not create duplicates or
bypass the allowlist.

## 4. Smoke test (staging)

1. Sign-in page shows the OIDC button labelled `OIDC_PROVIDER_NAME`; it is
   hidden while `OIDC_ISSUER_URL` is empty.
2. Full login round-trip against the IdP; first login creates the account
   (respecting §3) and lands in the workspace.
3. Second login reuses the same account.
4. With `OIDC_ALLOWED_GROUPS` set, a user outside the groups is rejected;
   a member logs in.
5. Unset all `OIDC_*` vars, restart: button disappears, email/Google login
   unaffected.

## 5. Rollback

OIDC is disabled by configuration, not by data changes:

```bash
# compose: clear OIDC_* from .env, then
docker compose -f docker-compose.selfhost.yml up -d backend
# helm: helm upgrade --reuse-values --set backend.config.oidcIssuerUrl="" ...
```

The `user_oidc_identity` table may be left in place — it is inert without the
env vars and future re-enablement re-links the same identities. Dropping it
is unnecessary and not recommended.
