# Identity, wallet, payments and notifications (stage E3)

How the account part of the platform works. The design is ported from the previous platform (`outegro-dev/platform`, its `auth-backend`, `wallet-backend`, `payments-backend` and `notifications-backend`). The new code follows the starters and this repo's [service contract](service-contract.md).

## Services

| Repo | Kind | Reached from | Owns |
|---|---|---|---|
| `id-web` | web | `https://id.nick-lukashik.com` | the only page the browser talks to: sign-in, account, sessions, wallet, payments, notifications |
| `id-api` | api | the cluster only | users, sign-in by email code, sessions, access tokens (ES256, JWKS), roles |
| `notifications-api` | api | the cluster only | email (sign-in codes, receipts), the in-app inbox |
| `wallet-api` | api | the cluster only | the GRM wallet: an append-only ledger, idempotent operations, internal debit and credit |
| `pay-api` | api | the cluster; providers reach only `/webhooks/*` | catalog, checkout, payment providers (`fake`, `lava`), webhooks |
| `packages` | library | — | `@nick-lukashik/auth` (token checks, Nest guards, permissions), `@nick-lukashik/contracts` (event payloads, money) |

Every service runs at least two replicas and keeps no state in process memory (service contract, "Replicas"). Each api has its own Postgres database. `id-api` also uses Redis (refresh tokens, rate limits). Events go through RabbitMQ with the outbox and inbox of `nest-example`.

## Tokens and the session

```
browser ──__Host-id_session──▶ id-web ──Bearer access token──▶ *-api ──JWKS──▶ id-api
```

- **Access token.** A JWT signed by `id-api` with ES256:
  - header: `typ: at+jwt`, `kid` = RFC 7638 thumbprint of the key;
  - claims: `iss` = `https://id.nick-lukashik.com`, `aud` = `nick-lukashik`, `sub` = user id, `sid` = session id, `roles`, `av` (access version);
  - lifetime 5 minutes.

  APIs verify it with `@nick-lukashik/auth`: JWKS from `http://id-api.prod.svc/.well-known/jwks.json`, cached and refetched on an unknown `kid`, with `iss`, `aud`, `typ`, expiry and only ES256 checked. Nothing else about the caller is trusted.
- **Refresh token.** Opaque `<sessionId>.<256 random bits>`, 30 days. Only a SHA-256 of it is stored, in Redis. Every refresh rotates it in one atomic Lua script:
  - the token issued just before is answered with the same new token for 20 s (two tabs, a lost response);
  - any other old token burns the whole session (reuse detection).
- **Session cookie.** `id-web` keeps `{ accessToken, accessExpiresAt, refreshToken, sessionId }` sealed with AES-256-GCM (key `SESSION_SECRET`, shared by the replicas) in one cookie:
  - `__Host-id_session`: `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, 30 days;
  - tokens never reach browser JavaScript;
  - the `id-web` proxy refreshes the access token when less than 60 s is left and rewrites the cookie;
  - a refused refresh clears the cookie;
  - a sign-in in progress (the challenge of step 1) waits in a second sealed cookie, `__Host-id_login`, for 10 minutes.

  The `id-web` side in detail: its README, "How sign-in works here".
- **Roles and permissions.** Roles live in `id-api` (`role_bindings`). Permissions are derived from roles by one table in `@nick-lukashik/auth`:
  - `owner` has every permission;
  - `support` has `users.read`, `sessions.revoke`, `notifications.read`.

  The first owner is granted from the command line (`node dist/cli/grant-owner.js <email>`). A paid purchase never grants a role.

## Flows

1. **Sign-in by email code.**
   1. `id-web` (Server Action) → `id-api POST /v1/login/email`.
   2. `id-api` stores an HMAC of a 6-digit code: 10 minutes, 5 attempts, 60 s before a resend, rate limits per email and per IP.
   3. `id-api` hands the code to `notifications-api POST /v1/internal/auth-codes` synchronously inside the cluster. The code never enters the broker, the outbox, a table of `notifications-api` or a log.
   4. The person types the code. `id-web` → `id-api POST /v1/login/email/verify`, which consumes the challenge atomically, creates the user on the first correct code and opens a session. `id-web` seals the tokens into the cookie.

   The response to step 1 is the same whether or not the account exists.
2. **Pages.** Server Components of `id-web` call the APIs over `http://<api>.prod.svc` with the access token of the session. Mutations are Server Actions (bodies capped at 1 MiB).
3. **Sign-out.** `id-web` → `id-api POST /v1/sessions/logout`, then the cookie is cleared. "Sign out everywhere else" revokes the other sessions of the user.
4. **Top-up.**
   1. On `/wallet` the person picks a GRM pack. `id-web` → `pay-api POST /v1/me/checkout` (`Idempotency-Key`) → the provider's checkout URL.
   2. The provider calls `pay-api /webhooks/<provider>`. `pay-api` checks the credential before reading the payload, then the amount and currency against the order snapshot.
   3. `pay-api` publishes `pay.payment.succeeded.v1` (outbox).
   4. `wallet-api` (inbox) credits the pack once, ref `pack:payment:<id>`, and publishes `wallet.credit.posted.v1`.
   5. `notifications-api` (inbox) adds an inbox entry and sends a receipt email.
5. **Service to service.** Only inside the cluster (NetworkPolicy `api-client`), with the receiving service's `INTERNAL_API_TOKEN` (`Authorization: Bearer`, compared in constant time):
   - `id-api` → `notifications-api` (sign-in codes);
   - `notifications-api` → `id-api` (email and language of a user);
   - later, roulette → `wallet-api` (stakes and payouts).

## Events

| Type | Producer | Consumers |
|---|---|---|
| `id.user.registered.v1` | id-api | notifications-api (welcome) |
| `pay.payment.succeeded.v1` | pay-api | wallet-api (credit), notifications-api (receipt) |
| `wallet.credit.posted.v1` | wallet-api | notifications-api (inbox) |

Payload schemas: `@nick-lukashik/contracts`. Envelopes, outbox, inbox, retries and DLQ: `nest-example` (`src/common/messaging`).

## Not in this stage yet

Each item is a later step with its own change:

- passkeys and Google sign-in;
- OAuth 2.1 authorization code with PKCE for other apps (landing, roulette), with silent sign-in through a hint cookie on the parent domain;
- `admin-web` on top of the permissions;
- the wallet reconciliation pass;
- a live check of the Lava adapter against the merchant account (offers, sandbox);
- Telegram as a notification channel;
- account deletion.

## Deploy prerequisites (gitops, infra)

- Postgres (CloudNativePG, one database per api), RabbitMQ and Redis in their own namespaces labelled `nick-lukashik.com/data: "true"`.
- An api variant without a public route (`components/internal`). `pay-api` gets one route, `hooks.nick-lukashik.com/webhooks/*`.
- Subdomains `id` and `hooks` in `active_stages` (`infra/terraform`), and the repos of stage `E3-core`.
- Sealed secrets:
  - `id-api`: the signing key (PKCS#8 ES256), the code pepper, the internal tokens;
  - `id-web`: `SESSION_SECRET`;
  - `notifications-api`: SMTP;
  - `pay-api`: the provider keys.
