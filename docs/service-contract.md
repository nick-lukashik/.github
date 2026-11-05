# Service runtime contract

What every service image provides, and what `gitops` and the `.github` workflows rely on. One source for the starters (`next-starter`, `nest-starter`) and the full references they copy from (`next-example`, `nest-example`), the `gitops` components and the reusable CI. A change here is a change in all of them.

Two kinds of service:
- **web** — a Next.js app (e.g. `landing-web`, `id-web`);
- **api** — a NestJS service (e.g. `id-api`, `pay-api`).

A scheduled job (e.g. `digests`) is not a service: no server, no port, no health endpoints. It has the same image rules, CI and deploy PR, builds with `kind: api`, and in `gitops` has its own CronJob and NetworkPolicy (`JOBS` in `scripts/check-apps.sh`).

## Image

| Item | Rule |
|---|---|
| Name | `ghcr.io/nick-lukashik/<repo-name>`, public |
| Tags | `sha-<7 chars>` and the full commit SHA. Deployed **by digest** only |
| Build args | `APP_VERSION=<full SHA>` only. Nothing environment-specific is baked in: no `NEXT_PUBLIC_*` value that differs between stage and prod. One image goes to stage and then to prod |
| Base | `node:<exact version>-trixie-slim`, readable tag in `FROM`, updated by Renovate |
| User | numeric `10001:10001` (`USER 10001:10001`). App files are owned by root and read-only to the process |
| Filesystem | works with `readOnlyRootFilesystem: true`. Writable paths are only `/tmp`, plus `/app/.next/cache` for web. Both are `emptyDir` in Kubernetes |
| Entrypoint | `node server.js` (web) or `node dist/main.js` (api); the process is PID 1 and handles `SIGTERM` |
| `HEALTHCHECK` | allowed for `docker run`; Kubernetes ignores it and uses the probes below |

## Runtime

| Item | Rule |
|---|---|
| HTTP port | the app listens on env `PORT`. `gitops` sets `PORT=3000` and names the container port `http`. Local defaults may differ (web 3000, api 4000) |
| Liveness | `GET /health/live` on `http`: the process answers, no dependency checks |
| Readiness | `GET /health/ready` on `http`: the service can do its core job. **api:** Postgres reachable (required). Redis and RabbitMQ, where a service uses them, are degradable (see Degradation) and only reported in the response body and metrics. **web:** the server can render; the API's state is reported, not required, so pages that need no API keep working. Returns 503 when not ready |
| Version | both health endpoints return header `x-app-version: <APP_VERSION>`. The post-deploy smoke check waits for it |
| Logs | JSON lines on stdout, field `requestId` when there is a request. No secrets |
| Replicas | at least two (gitops: HorizontalPodAutoscaler 2 to 4 on CPU, PodDisruptionBudget of one), so a service is stateless: sessions, caches and rate limits live in Postgres or Redis, never in process memory; scheduled and background work runs through pg-boss or a database lock, so two pods never do it twice; nothing assumes it is the only pod |
| Shutdown | on `SIGTERM`: stop accepting, finish in-flight requests and consumers, close pools, exit within 25 s. Pods get `preStop` sleep 5 s and `terminationGracePeriodSeconds: 30` |
| Config | non-secret env from ConfigMap `<svc>-config`; secrets from SealedSecret `<svc>-secrets`, both via `envFrom`. Validated with Zod at start; a bad config fails fast |
| Indexing (web) | runtime env `ROBOTS=index\|noindex` (default `index`): `robots.txt` and the robots meta tag read it per request |
| Request size | at most 1 MiB per request body on every public route. api: Traefik middleware `body-limit` at the edge plus the body parser. web: in the app, because Traefik buffering also buffers whole responses and would break streamed HTML; the `/api` proxy caps the body while reading it, also without `Content-Length`; Server Actions are capped by `experimental.serverActions.bodySizeLimit: '1mb'` in `next.config.ts` (the language switch is one); the app has no other body-accepting routes. Larger uploads need a separate streaming route designed for them |
| Time budgets | api: whole request 10 s; SQL `statement_timeout` 5 s, `lock_timeout` 2 s, `idle_in_transaction_session_timeout` 10 s; waiting for a pool connection 2 s; Redis command 500 ms (on timeout discard the connection and reconnect without replay; queue at most 256 commands). SQL client reply deadline 6 s closes the stuck connection, including inside transactions. Migrations: a lock wait of at most 1 s (below the app's 2 s, so queries queued behind a waiting DDL still get through), the whole run rolled back and retried up to 10 times. web: reading a proxied request body 10 s, upstream call 10 s. A dependency that hangs fails the request fast instead of filling the pool |

### api only
| Item | Rule |
|---|---|
| Metrics | Prometheus text on port `9464` (env `METRICS_PORT`), path `/metrics`, container port name `metrics`. Never on the public `http` port |
| Migrations | `node dist/db/migrate.js`: applies pending migrations, idempotent, exit 0. Runs as an Argo CD `Sync`-hook Job in wave 0 with the same image, config and secrets: after `<svc>-secrets` (wave -1), before the Deployment (wave 1). Not `PreSync`: on the first sync and on every config change the hashed ConfigMap does not exist yet during PreSync |
| Callers | accepts HTTP only from Traefik and from pods labelled `nick-lukashik.com/api-client: "true"` in namespace `prod` (NetworkPolicy). An api that calls another api adds that label in its own app folder |
| Degradation | Applies to what a service uses (the starter has only Postgres). Postgres down: not ready. Redis down: cache is bypassed, the rate limiter fails open (logged and counted). RabbitMQ down: writes still succeed, events wait in the outbox, consumers resubscribe when the broker returns. Each state is a metric with an alert |
| Messaging | publish with `mandatory: true`; an unroutable (returned) message is a delivery failure and its outbox row stays unpublished. Quorum queues use `x-dead-letter-strategy: at-least-once` with `x-overflow: reject-publish`. A consumer cancelled by the broker or a closed channel is re-established without waiting for a reconnect; required consumers are reported in health and metrics. A DLQ replay command exists |

### web only
| Item | Rule |
|---|---|
| Metrics | none; HTTP metrics come from Traefik |
| Migrations | none |
| API access | pod label `nick-lukashik.com/api-client: "true"`; server-side calls go to `http://<api>.prod.svc` via env `API_URL` |
| Site URL | env `SITE_URL` (canonical, sitemap, Open Graph). gitops fills it from the route host, so the domain lives only in gitops settings |

## gitops

- Namespace `prod`: default-deny **ingress and egress**.
- `components/service`: common to both kinds:
  - Deployment (no `replicas`: a HorizontalPodAutoscaler keeps 2 to 4, and a PodDisruptionBudget lets one pod go at a time), Service, IngressRoute with middleware `real-ip` (`components/api` adds `body-limit`);
  - egress to cluster DNS only (kinds add the rest);
  - probes: startup, liveness and readiness;
  - security context: uid 10001, `seccompProfile: RuntimeDefault`, drop `ALL`, `allowPrivilegeEscalation: false`, read-only root filesystem, `automountServiceAccountToken: false`;
  - `/tmp` `emptyDir`, `preStop` sleep, resources, `priorityClassName: service`;
  - NetworkPolicy allowing ingress from Traefik.
- `components/api`: adds the migrate Job (Sync hook, wave 0), the `metrics` port, a ServiceMonitor on it, a NetworkPolicy for api clients and Prometheus, and egress to the data stores (namespaces labelled `nick-lukashik.com/data: "true"`, stage 3) and to public addresses on 443 (private ranges excluded).
- `components/web`: adds the `/app/.next/cache` `emptyDir`, the `api-client` label, and egress to api pods on `http`.
- Monitoring (`monitoring`, `monitoring-node`): ingress allowed only from the components that need it (e.g. Loki only from Alloy and Grafana).
- A service folder `apps/<svc>/prod/kustomization.yaml` includes `service` plus exactly one of `api` or `web`, sets the host, and sets the image as an `images:` entry named `ghcr.io/nick-lukashik/<svc>` with a `digest` and no `newName`. The deploy PR changes only that digest.

## CI (`nick-lukashik/.github`)

- Reusable workflows are referenced by the moving major tag `@v1`, never `@master`. Breaking changes go to `v2`.
- `gitops` is a public repository so that its `master` ruleset (PR, one approval, no bypass for bots or GitHub Apps) is enforced on the GitHub Free plan. It holds no plain secrets (Sealed Secrets only).

- **`build.yml`:** `kind: web` or `kind: api`, `services`, `publish`; nothing else is configurable. In one job it:
  - runs `pnpm verify`, for the api with the CI service containers the repo lists (`postgres`, `redis`, `rabbitmq`), exported as `TEST_*_URL`;
  - for the web, runs `pnpm test:e2e` (Playwright) on that same build;
  - builds the image from `Dockerfile` with `APP_VERSION` as the only build arg;
  - on `master`, a second job pushes to GHCR and outputs the `digest`.
  - A change to docs only (`docs/`, `.claude/`, root `*.md`) skips all of it: no image, no deploy PR.
- **`deploy-pr.yml`:** opens or updates the PR in `gitops` that sets the new digest. The owner's merge is the deploy.
- **`smoke.yml`:** after sync, polls `https://<host>/health/ready` until `x-app-version` equals the SHA, then checks the read-only paths in the route annotation `nl-gitops/smoke-paths` (space-separated). Defaults: web `/`, api `/openapi.json`; service folders override them with functional checks, including a backend read for API-dependent web apps. It reads the host from the service's gitops route (annotation `nl-gitops/host`), not from a hard-coded URL.
- **Service repo:** `.github/workflows/ci.yml` calls these reusable workflows with secrets passed explicitly, never `secrets: inherit`. `deploy-pr.yml` gets `DEPLOYER_APP_ID`, `DEPLOYER_PRIVATE_KEY` and optionally `OPS_TELEGRAM_TOKEN`. `publish: ${{ !github.event.repository.is_template }}` in starters and services, `publish: false` in the examples (never deployed).
