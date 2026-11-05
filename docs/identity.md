# Identity, wallet, payments and notifications (stage E3)

How the account part of the platform works. The design is ported from the previous platform (`outegro-dev/platform`, its `auth-backend`, `wallet-backend`, `payments-backend` and `notifications-backend`). The new code follows the starters and this repo's [service contract](service-contract.md).

## Services

| Repo | Kind | Reached from | Owns |
|---|---|---|---|
| `id-web` | web | `https://id.nick-lukashik.com` | the only account page the browser talks to: sign-in, account, sessions, wallet, notifications, and sign-in for the other sites (`/authorize`). Coin packs and the payment pages only with `PAYMENTS_ENABLED`: off in prod, where `/payments` answers 404 |
| `id-api` | api | the cluster only | users, sign-in (email code, Google, passkeys), sessions, access tokens (ES256, JWKS), sign-in codes for the other sites, roles |
| `notifications-api` | api | the cluster only | email through Resend (sign-in codes, receipts), the in-app inbox |
| `wallet-api` | api | the cluster only | the coin wallet: an append-only ledger, idempotent operations, internal debit and credit |
| `pay-api` | api | not deployed yet; then the cluster, and providers reach only `/webhooks/*` | catalog, checkout, payment providers (`fake`, `lava`), webhooks |
| `landing-web` | web | `https://nick-lukashik.com` | the portfolio; signs in through `id-web` and keeps its own session |
| `packages` | library | — | `@nick-lukashik/auth` (token checks, Nest guards, roles and permissions), `@nick-lukashik/contracts` (event payloads, coins, money) |

Every service runs at least two replicas and keeps no state in process memory (service contract, "Replicas"). Each api has its own Postgres database. `id-api` also uses Redis (refresh tokens, passkey challenges, rate limits). Events go through RabbitMQ with the outbox and inbox of `nest-example`.

## Sign-in methods

All three end in the same kind of session. The email code always works, so removing a passkey or unlinking Google never locks anyone out.

| Method | How | Notes |
|---|---|---|
| Email code | 6 digits, 10 minutes, 5 attempts; the account is created by the first correct code | the anchor: every account has a verified email |
| Google | OpenID Connect, authorization code with PKCE and a nonce. `id-web` keeps state, verifier and nonce in a sealed cookie for 10 minutes; `id-api` exchanges the code (it holds the client secret) and checks the ID token (Google's JWKS, `iss`, `aud`, `nonce`, `email_verified`) | a new Google account with a new email gets a new user. An email that already has an account is never linked by itself (`link_required`): the owner signs in with the code and links Google on the account page, which needs a fresh sign-in |
| Passkey | WebAuthn with SimpleWebAuthn. RP ID `nick-lukashik.com`, ceremonies accepted only from `https://id.nick-lukashik.com`, user verification required, discoverable credentials (no email asked), one-time challenges in Redis for 5 minutes, ES256 / EdDSA / RS256 only | adding one needs a fresh sign-in. A signature counter that does not grow refuses the sign-in (a copied key) |

**Fresh sign-in.** Adding a passkey and linking Google need an `id-web` session signed in at most 15 minutes ago (`PASSKEY_FRESH_SIGN_IN_MS`); a session of another site never qualifies. Otherwise `id-api` answers 403 `reauthentication_required` and the person signs in again, so a stolen older session cannot attach a key or a Google account of its own.

The email field offers passkeys in its autofill (`autocomplete="username webauthn"`). The Google button is a link, not a form: CSP `form-action` would stop a form's redirect to Google before JavaScript loads.

## Tokens and the session

```
browser ──__Host-id_session──▶ id-web ──Bearer access token──▶ *-api ──JWKS──▶ id-api
```

- **Access token.** A JWT signed by `id-api` with ES256:
  - header: `typ: at+jwt`, `kid` = RFC 7638 thumbprint of the key;
  - claims: `iss` = `https://id.nick-lukashik.com`, `aud` = `nick-lukashik`, `sub` = user id, `sid` = session id, `roles`, `av` (access version);
  - lifetime 5 minutes.

  APIs verify it with `@nick-lukashik/auth`: JWKS from `http://id-api.prod.svc/.well-known/jwks.json`, cached and refetched on an unknown `kid`, with `iss`, `aud`, `typ`, expiry and only ES256 checked. Nothing else about the caller is trusted.
- **Revocation latency.** A session ended elsewhere stops at once where a person sees it: `id-web` (`requireSession`, on a page behind sign-in or an action) and a site (its proxy, on a page visit) ask `id-api` `GET /api/v1/me/session`, which answers from Redis. If `id-api` cannot tell, the page goes on by its token, as in an outage. The APIs do not ask: they check the token offline, and tokens stay on the servers. Nothing compares `av` yet: `id-api` bumps it on every role or grant change, for a later check. So a role or grant taken away or a suspension reaches the APIs only when the access token already issued expires, up to 5 minutes later. `id-api` acts on the next refresh: a revoked session or a suspended account gets no new token, and a new token carries the roles and grants of that moment.
- **Refresh token.** Opaque `<sessionId>.<256 random bits>`, 30 days. Only a SHA-256 of it is stored, in Redis. Every refresh rotates it in one atomic Lua script:
  - the token issued just before is answered with the same new token for 20 s (two tabs, parallel requests of one page, a lost response);
  - `REUSE`: any other superseded token, while it is among the session's newest 512 (`rtp:<sid>`), burns the whole session (reuse detection);
  - `DEAD`: anything else (revoked, expired, never issued, or superseded so long ago that the cap dropped it) gets a 401 and changes nothing, so a session id alone burns nothing.
- **Session cookie.** `id-web` keeps `{ accessToken, accessExpiresAt, refreshToken, sessionId }` sealed with AES-256-GCM (key `SESSION_SECRET`, shared by the replicas) in one cookie:
  - `__Host-id_session`: `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/`, 30 days;
  - tokens never reach browser JavaScript;
  - the `id-web` proxy refreshes the access token when less than 60 s is left and rewrites the cookie;
  - a refused refresh clears the cookie;
  - a sign-in in progress waits in a second sealed cookie: `__Host-id_login` (the email code, 10 minutes) or `__Host-id_google` (Google, 10 minutes).

  The `id-web` side in detail: its README, "How sign-in works here".
- **Sessions per site.** Every site has its own session (`sessions.client_id`: null for `id-web`, `landing-web` for the portfolio), a child of the `id-web` session it signed in from (`sessions.parent_session_id`).
- **Sign-out is for the whole account.** "Sign out" on any site signs out of every site in this browser: the site ends its own session and sends the browser to `id.nick-lukashik.com/logout?return=<the site>`; `id-web` ends its session, and id-api ends its children with it; the hint goes, and a site's session ends as soon as the hint is gone. "Sign out everywhere else" on the account page ends the other devices and keeps this browser's sites.
- **The hint.** While its session lives, `id-web` keeps the cookie `nl_hint=1` on the parent domain (`nick-lukashik.com`, `HttpOnly`). It is not a credential: it only tells the other sites to try a silent sign-in.

## Roles and grants

Two different things, both in the access token.

- **Roles** are for staff, granted by hand, never bought (`role_bindings`). Permissions come from roles through one table in `@nick-lukashik/auth`:

  | Role | Granted | Permissions |
  |---|---|---|
  | `owner` | by hand: `node dist/cli/grant-owner.js <email>` for the first one | all |
  | `support` | by hand | `users.read`, `sessions.revoke`, `notifications.read` |

- **Grants** are what a person paid for: a plan of a service, `service:plan` (`roulette:vip`, `edu:pro`). Any number at once, one per subscription, each for its own paid period (`grants`). Services check `hasGrant(caller, 'roulette', 'vip')`. A one-off coin pack never gives a grant.
  - pay-api will publish `pay.subscription.changed.v1` (`subscriptionId`, `userId`, `service`, `plan`, `accessUntil` or null, `version`). id-api already consumes it (queue `id-api.subscriptions`): `accessUntil` sets the end, null takes the grant away now (a refund), and a version not newer than the last one applied changes nothing.
  - **No job ends a grant.** Access tokens carry only the grants in force when they are issued, and a token lives 5 minutes: when a period runs out, the next token simply lacks it. A service that must be exact to the second compares nothing itself either: it trusts the token, which is at most 5 minutes old.
  - pay-api sells no subscriptions yet.

## Flows

1. **Sign-in by email code.**
   1. `id-web` (Server Action) → `id-api POST /api/v1/login/email`.
   2. `id-api` stores an HMAC of a 6-digit code: 10 minutes, 5 attempts, 60 s before a resend, rate limits per email and per IP.
   3. `id-api` hands the code to `notifications-api POST /api/v1/internal/auth-codes` synchronously inside the cluster. The code never enters the broker, the outbox, a table of `notifications-api` or a log.
   4. The person types the code. `id-web` → `id-api POST /api/v1/login/email/verify`, which consumes the challenge atomically, creates the user on the first correct code and opens a session. `id-web` seals the tokens into the cookie.

   The response to step 1 is the same whether or not the account exists.
2. **Sign-in with Google.**
   1. The link `/login/google` asks `id-api GET /api/v1/login/google/config`, seals state, nonce and the PKCE verifier into `__Host-id_google` and sends the browser to Google.
   2. Google returns to `/login/google/callback`. The state must match; `id-web` → `id-api POST /api/v1/login/google` with the code, the verifier and the nonce.
   3. `id-api` exchanges the code at Google, checks the ID token, finds the user by the Google account (`identities`) or creates one, and opens a session.

   Linking from the account page is the same round trip with `intent=link`, ending in `POST /api/v1/me/identities/google`.
3. **Sign-in with a passkey.** `id-web` (Server Action) → `id-api POST /api/v1/login/passkey/options` → the browser asks the device → `POST /api/v1/login/passkey/verify`. Adding one: `POST /api/v1/me/passkeys/options`, then `POST /api/v1/me/passkeys`.
4. **Sign-in on another site (the portfolio).** Authorization code with PKCE between first-party sites, not a public OIDC server. "Silent sign-in" below means: signed in at `id-web`, signed in on the portfolio too, with no click.
   1. "Sign in" on `nick-lukashik.com` → its `/auth/sign-in` seals state, verifier and the return path into `__Host-site_sso` and sends the browser to `id.nick-lukashik.com/authorize`.
   2. `id-web` checks the site and its exact callback (`id-api GET /api/v1/oauth/clients/:id`), signs the person in if needed (`/login?continue=/authorize…`), then asks `id-api POST /api/v1/oauth/authorize` for a code: 60 seconds, once, bound to the site, the callback and the PKCE challenge. The browser goes back to `https://nick-lukashik.com/auth/callback?code=…&state=…`.
   3. The portfolio checks the state and exchanges the code server to server: `id-api POST /api/v1/oauth/token` with the verifier. The result is its own session (`client_id = landing-web`), sealed into `__Host-site_session` with its own `SESSION_SECRET`.
   4. Its proxy refreshes that session like `id-web` does, and asks `GET /api/v1/me` again then (a new display name shows).
   5. **Silent sign-in.** A visitor the portfolio has no session for, with the hint, goes through `/auth/sign-in?silent=1` once: `/authorize?prompt=none` answers with a code when signed in at `id-web`, `login_required` otherwise, and shows no page. One attempt per 5 minutes at most; never for crawlers, prefetches or Lighthouse.
5. **Pages.** Server Components of `id-web` call the APIs over `http://<api>.prod.svc` with the access token of the session. Mutations are Server Actions (bodies capped at 1 MiB).
6. **Sign-out.** `id-web` → `id-api POST /api/v1/sessions/logout` with the refresh token itself, which ends the session and its children (the sites signed in from it); then the cookie and the hint are cleared. Only the session's current refresh token (or the one before it, within the 20 s grace) ends it: a session id alone ends nothing. From another site: its `/auth/sign-out`, then `id-web` `/logout?return=…`. "Sign out everywhere else" revokes the user's sessions on other devices, with their children.
7. **Top-up.** Built, not live: `pay-api` is not deployed and `PAYMENTS_ENABLED` is unset in prod, so `/wallet` offers no packs and no money moves.
   1. On `/wallet` the person picks a coin pack. `id-web` → `pay-api POST /api/v1/me/checkout` (`Idempotency-Key`) → the provider's checkout URL.
   2. The provider calls `pay-api /webhooks/<provider>`. `pay-api` checks the credential before reading the payload, then the amount and currency against the order snapshot.
   3. `pay-api` publishes `pay.payment.succeeded.v1` (outbox).
   4. `wallet-api` (inbox) credits the pack once, ref `pack:payment:<id>`, and publishes `wallet.credit.posted.v1`.
   5. `notifications-api` (inbox) adds an inbox entry and sends a receipt email.
8. **Service to service.** Only inside the cluster (NetworkPolicy `api-client`), with the receiving service's `INTERNAL_API_TOKEN` (`Authorization: Bearer`, compared in constant time):
   - `id-api` → `notifications-api` (sign-in codes);
   - `notifications-api` → `id-api` (email and language of a user);
   - `pay-api` → `id-api` (at checkout: the buyer's email, language and status, `GET /api/v1/internal/users/:id`);
   - `pay-api` → `wallet-api` (which refs the ledger has, for its reconciliation, `POST /api/v1/internal/refs`);
   - later, roulette → `wallet-api` (stakes and payouts).

## Coins

The wallet's one unit is the coin (`COIN`): whole numbers, no cash-out, no transfers between people. Packs (`pay-api`, `src/catalog/catalog.ts`):

| Pack | USD | EUR | RUB |
|---|---|---|---|
| 100 coins | 1 | 1 | 100 |
| 550 coins (+10%) | 5 | 5 | 500 |
| 1200 coins (+20%) | 10 | 10 | 1000 |

A price change applies to new checkouts; a payment keeps its snapshot.

## Events

| Type | Producer | Consumers |
|---|---|---|
| `id.user.registered.v1` | id-api | notifications-api (welcome) |
| `pay.payment.succeeded.v1` | pay-api | wallet-api (credit), notifications-api (receipt) |
| `wallet.credit.posted.v1` | wallet-api | notifications-api (inbox) |
| `pay.subscription.changed.v1` | pay-api (once it sells subscriptions) | id-api (the grant of the plan) |

Payload schemas: `@nick-lukashik/contracts`. Envelopes, outbox, inbox, retries and DLQ: `nest-example` (`src/common/messaging`).

## Not in this stage yet

Each item is a later step with its own change:

- subscriptions in `pay-api` (the grant side in `id-api` is ready);
- `admin-web` on top of the permissions;
- the wallet's own reconciliation pass (balance drift, chain breaks, zero sum; `wallet-api` README). `pay-api`'s check of payments against the ledger is built;
- the choice of a payment provider, and a live check of its adapter (`pay-api` is provider-neutral, Lava is frozen as a second implementation);
- the WebAuthn Signal API (the device forgets a passkey removed here);
- Telegram as a notification channel;
- account deletion by the person (until then, on request: `gitops/docs/runbook/accounts.md`).

## Deploy prerequisites (gitops, infra)

- **Data stores.** One Postgres cluster (CloudNativePG) with a database and a role per api (`id`, `notifications`, `wallet`, `pay`); base backups and WAL to object storage (Cloudflare R2) for point-in-time recovery. RabbitMQ and Redis. All three in namespace `data`, labelled `nick-lukashik.com/data: "true"`. No PgBouncer at first: each pod keeps a small pool (`DATABASE_POOL_MAX=5`), and CloudNativePG's `Pooler` (PgBouncer, transaction mode) is added when the connections no longer fit.
- An api variant without a public route (`components/internal`). `pay-api` gets one route, `hooks.nick-lukashik.com/webhooks/*`.
- Subdomains `id` and `hooks` in `active_stages` (`infra/terraform`), and the repos of stage `E3-core`.
- **Google.** An OAuth client "Web application" in Google Cloud Console, redirect URI `https://id.nick-lukashik.com/login/google/callback`, scopes `openid email profile`.
- Configuration:
  - `id-api`: `GOOGLE_REDIRECT_URI`, `WEBAUTHN_RP_ID=nick-lukashik.com`, `WEBAUTHN_ORIGIN=https://id.nick-lukashik.com`, `OAUTH_CLIENTS=landing-web=https://nick-lukashik.com/auth/callback`;
  - `id-web`: `FORM_ACTION_ORIGINS=https://nick-lukashik.com` (plus the checkout origin of the provider, once one is chosen and payments are on);
  - `landing-web`: `ID_URL=https://id.nick-lukashik.com`, `ID_API_URL=http://id-api.prod.svc`, and egress to `id-api` (NetworkPolicy `api-client`). Without them the portfolio stays as it is, with no accounts.
- Sealed secrets:
  - `id-api`: the signing key (PKCS#8 ES256), the code pepper, the internal tokens, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`;
  - `id-web`: `SESSION_SECRET`;
  - `landing-web`: its own `SESSION_SECRET`;
  - `notifications-api`: the Resend SMTP key (domain `nick-lukashik.com` verified in Resend, its DNS records in `infra/terraform/cloudflare`);
  - `pay-api`: the provider keys.
