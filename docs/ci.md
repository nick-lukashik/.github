# CI/CD of services

Reusable workflows of the `nick-lukashik` org. A service repo calls them by tag, as
`nick-lukashik/.github/.github/workflows/<file>@v1` or `build.yml@v1.1.0` for a build that
needs `github-packages`, and has no build steps of its own. A change here reaches a service
when a tag it pins includes that change ([Versions](#versions-v1-and-v2)), not when it is
merged to `master`.

They implement the CI part of the service runtime contract ([service-contract.md](service-contract.md)): `pnpm verify`, one image per commit with `APP_VERSION` as the only
build arg, deploy by digest through a PR in `gitops`, smoke check against the host of
the service's route.

| File | What it does |
|---|---|
| [`build.yml`](../.github/workflows/build.yml) | `pnpm verify` (with the service's test containers for an api, Playwright e2e for a web), image to GHCR, digest as output |
| [`deploy-pr.yml`](../.github/workflows/deploy-pr.yml) | PR in `gitops` with the new digest, opened by a GitHub App |
| [`smoke.yml`](../.github/workflows/smoke.yml) | checks prod after a deploy, result to Telegram |
| [`ci.yml`](../.github/workflows/ci.yml), [`actionlint.yml`](../.github/workflows/actionlint.yml) | CI of this repo: actionlint with shellcheck, examples included |

Examples to copy:

| File | Goes to | What |
|---|---|---|
| [`examples/ci-web.yml`](examples/ci-web.yml) | `.github/workflows/ci.yml` of a web (Next.js) | `build.yml` (e2e included), `deploy-pr.yml` |
| [`examples/ci-api.yml`](examples/ci-api.yml) | `.github/workflows/ci.yml` of an api (NestJS) | `build.yml` with the test services, `deploy-pr.yml` |
| [`examples/smoke.yml`](examples/smoke.yml) | `.github/workflows/smoke.yml` of a service | `smoke.yml` on `deployment_status` or by hand |
| [`examples/gitops-smoke.yml`](examples/gitops-smoke.yml) | `.github/workflows/smoke.yml` of `gitops` (installed) | smoke after every merged deploy PR, until Argo CD reports deployments |

Starters, examples, the services copied from them, how fixes are ported and the plan for packages: [templates.md](templates.md).

## Versions: v1 and v2

Callers reference a tag, `@v1` or an exact version, never `@master` and never a branch.
`master` of this repo can then change without changing every service's CI at the same
moment, and a breaking change cannot reach a caller that did not opt in.

- **The tags today.** `v1` and `v1.0.0` are still the first commit. `v1.1.0` adds the input
  `github-packages` to `build.yml`, its only change to the workflows, and `v1` was not moved
  to it. So the builds that need that input (`id-api`, `notifications-api`, `wallet-api`,
  `pay-api`) call `build.yml@v1.1.0`; every other call, their `deploy-pr.yml` included, is
  on `@v1`.
- **Compatible change** (a new optional input, a fix, a new pinned action version): merge
  to `master` and tag the merge commit with the next exact version. A caller gets it by
  changing its pin to that tag. The callers of `@v1` get it when `v1` is moved to it, which
  has not been done yet:

  ```sh
  git switch master && git pull --ff-only
  git tag -a v1.2.0 -m "v1.2.0: <what changed>"      # exact versions are never moved
  git tag -f -a v1 -m "v1 -> v1.2.0" v1.2.0^{}       # the moving major
  git push origin v1.2.0
  git push -f origin v1
  ```

  Moving `v1` does not reach an exact pin such as `build.yml@v1.1.0`. After a move, check one
  caller of each kind (a web and an api build, a smoke run) on the new `v1`. Going back is
  moving `v1` to the previous exact tag.
- **Breaking change** (a required input, a renamed output or check, changed permissions or
  secrets): tag `v2.0.0` and `v2`; `v1` stays where it is. Callers move to `@v2` one by
  one, each in its own PR; `v1` gets only security fixes until the last caller has moved.
- Only the organization owner can create, move or delete `v*` tags: ruleset `release-tags`
  of this repo (`infra`, `terraform/github/rulesets.tf`). Moving `v1` changes the CI of every
  caller of `@v1`, with access to their `packages: write` tokens and the deployer secrets.
- The required check `ci / verify` of the callers comes from job `verify` of `build.yml`.
  Renaming that job is a breaking change.

**Renovate.** Service repos pin third-party actions by commit SHA, and Renovate keeps those
pins current (`helpers:pinGitHubActionDigests`). The reusable workflows of this repo are
the exception: they stay on their tag, not on a SHA, so that moving `v1` reaches the callers
of `@v1`. In `renovate.json` of a service repo:

```json
{
  "packageRules": [
    {
      "matchManagers": ["github-actions"],
      "matchPackageNames": ["nick-lukashik/.github"],
      "pinDigests": false
    }
  ]
}
```

For `@v1`, Renovate then opens a PR only when a new major tag (`v2`) appears; that PR is
the migration to the breaking version and is reviewed as such.

## How it works

```mermaid
sequenceDiagram
  participant Repo as Service repo
  participant GA as GitHub Actions
  participant GHCR
  participant App as deployer (GitHub App)
  participant G as gitops
  participant Argo as Argo CD
  Repo->>GA: PR: ci / verify (pnpm verify, e2e for a web, docker build without push)
  Note over Repo: the owner merges the PR (squash)
  Repo->>GA: push to master: ci / verify, ci / image
  GA->>GHCR: ghcr.io/nick-lukashik/<image>:sha-<sha7>, digest
  GA->>App: token for 1 hour, gitops only
  App->>G: PR "deploy <service> <sha7>" with the new digest
  Note over G: the owner merges the PR = the deploy
  G->>Argo: apps/<service>/prod changed
  Argo->>Repo: deployment production = success (later)
  Repo->>GA: smoke against https://<route host>, result to Telegram "Deploys"
```

1. On a PR, job `verify` installs dependencies, runs `pnpm verify` (a web then runs
   `pnpm test:e2e` on that build) and builds the image without pushing it. The check is
   called `ci / verify`; the `master` ruleset requires it. A change to docs only skips all
   of it and passes.
2. After the merge to `master`: the same `verify`, then job `image` pushes the image and
   outputs its digest. A docs-only commit pushes nothing, so no deploy PR follows.
3. Job `promote` (`deploy-pr.yml`) gets a token of the GitHub App `nick-lukashik-deployer`,
   changes the digest in `gitops` and opens a PR. An older open deploy PR of the same
   service is closed with the comment "Superseded by #N."
4. Merging the PR in `gitops` is the deploy. Argo CD syncs `apps/<service>/prod`.
5. `smoke.yml` waits until prod serves the new version, checks pages and runs the
   service's `pnpm smoke`. Green and red results go to the Telegram topic "Deploys".

## Connect a service

1. Copy [`examples/ci-web.yml`](examples/ci-web.yml) or [`examples/ci-api.yml`](examples/ci-api.yml)
   to `.github/workflows/ci.yml`. The id of the calling job must be `ci`: that gives the
   check `ci / verify`. Secrets are passed by name, never `secrets: inherit`.
2. Copy [`examples/smoke.yml`](examples/smoke.yml) to `.github/workflows/smoke.yml`. No URL
   to set: it comes from the route of the service in `gitops`.
3. Give the repo access to the org secrets `DEPLOYER_APP_ID`, `DEPLOYER_PRIVATE_KEY`,
   `OPS_TELEGRAM_TOKEN` (**Selected repositories** of each secret).
4. `services` lists only what the service's tests use (nest-starter: `postgres`).
5. In `gitops`, create `apps/<service>/prod` from `apps/_example-web` or `apps/_example-api`
   (`apps/README.md` of `gitops`), with the digest of the first image. See the
   [convention](#digest-convention-in-gitops).
6. The Dockerfile is `Dockerfile` at the repo root and follows the contract: `ARG APP_VERSION` / `ENV APP_VERSION` in the final
   stage, user `10001:10001`, a read-only root filesystem ([version convention](#version-convention)).
7. After the first push of the image, check the visibility of the package
   `ghcr.io/nick-lukashik/<image>`: it must be **Public**, so the cluster pulls it without a
   secret. Set once in **Package settings → Change visibility**.
8. A repo made from a starter records it in `.template-version` ([templates.md](templates.md)).
   So far only `landing-web` has one.

### Template repos

The starters (`next-starter`, `nest-starter`) call `build.yml` with
`publish: ${{ !github.event.repository.is_template }}`, like `examples/ci-*.yml`:

- in the template repo itself `is_template` is true: verify runs, nothing is pushed,
  job `deploy` is skipped;
- a repo created from the template is not a template: it publishes and deploys
  without edits to `ci.yml`.

This needs the starter repos to be marked as templates (**Settings → Template
repository**); the Terraform in `infra` sets `is_template` for them.

The reference repos `next-example` and `nest-example` are not templates. They pass
`publish: false` and have no `deploy` job: their CI only proves that every capability
works together (all three test containers for `nest-example`).

## build.yml

Job `verify` runs on any event. Job `image` runs only when all of these hold: input
`publish` is true, the event is `push` or `workflow_dispatch`, the branch is the
default branch, and the commit changes more than docs.

### Inputs

| Input | Default | What it sets |
|---|---|---|
| `kind` | required | `web` (Next.js: `.next/cache` is cached, `pnpm test:e2e` runs after `pnpm verify`) or `api` (NestJS, or a scheduled job: `pnpm verify`, then the image) |
| `services` | empty | test containers, comma-separated: `postgres`, `redis`, `rabbitmq`; only what the service uses |
| `publish` | `true` | `false` never pushes; starters pass `${{ !github.event.repository.is_template }}`, examples `false` |
| `github-packages` | `false` | `true`: pnpm and the image build authenticate to GitHub Packages for `@nick-lukashik/*` (see "Registry token"). Not in `@v1`: such a build calls `build.yml@v1.1.0` |

Everything else is fixed by the contract: the image is `ghcr.io/nick-lukashik/<repo name>`,
Node comes from `.node-version`, the image is built from `Dockerfile` at the repo root
for `linux/amd64` with `APP_VERSION=<full SHA>` as the only build arg. An input for
another case (another registry, build args, a second Dockerfile) is added when a real
service needs it, as a compatible `v1` change ([Versions](#versions-v1-and-v2)).

Secrets: none. `GITHUB_TOKEN` is enough (GHCR push).

### Outputs

| Output | Example | Empty when |
|---|---|---|
| `image` | `ghcr.io/nick-lukashik/landing-web` | the image was not pushed |
| `digest` | `sha256:…` (64 hex) | the image was not pushed |
| `tag` | `sha-1a2b3c4` | the image was not pushed |

### Test services

`services` starts the containers below, with throwaway credentials, and exports the URLs
under the names `test/global-setup.ts` of the nest repos reads instead of starting
Testcontainers. Each service lists only what it uses: a nest-starter service only
`postgres`, nest-example all three.

| Service | Image | Exported |
|---|---|---|
| `postgres` | `postgres:18.6-alpine` | `TEST_DATABASE_URL=postgres://app:app@localhost:5432/app` |
| `redis` | `redis:8.10.3-alpine` | `TEST_REDIS_URL=redis://localhost:6379` |
| `rabbitmq` | `rabbitmq:4.4.0-alpine` | `TEST_RABBITMQ_URL=amqp://app:app@localhost:5672` |

The versions are exact and match `compose.yaml` and `test/containers.ts` of the nest
repos; Renovate updates them in both places. Failure tests that stop and start a
dependency (nest-example) start their own containers, so Docker is used in CI too.

### Details

- **pnpm and Node.** pnpm is installed in the version of `packageManager` in
  `package.json`, with the store cache. Node comes from `.node-version`.
- **Build args.** `--build-arg APP_VERSION=<full SHA>` only. Nothing environment-specific
  (no `NEXT_PUBLIC_*` value that differs between environments) is baked in. Today the image
  goes to prod alone; once stage is built (`E7-stage`) the same image goes there first.
- **Registry token.** Only with `github-packages: true` (services that use
  `@nick-lukashik/*`): a user-level npmrc with the placeholder `${NPM_TOKEN}` and the run's
  `GITHUB_TOKEN` in the env of `pnpm install` only, as in `smoke.yml` (pnpm ignores
  credentials with variables in a repository's `.npmrc`). The image build gets a file with
  the token as the BuildKit secret `npmrc`; the Dockerfile mounts it for `pnpm install`
  only (`RUN --mount=type=secret,id=npmrc,target=/root/.npmrc,required=false`), so it is in
  no layer. `verify` has `packages: read` for this; without the input nothing is written.
- **Image.** Tags `sha-<sha7>` and the full commit SHA; deployed by digest only.
  `provenance: true`, `sbom: true`. OCI labels (`org.opencontainers.image.source`,
  `revision`) link the package to the repo. Layer cache: GitHub Actions cache, written on
  `master` (`mode=max`), read-only on PRs.
- **Concurrency.** `verify` on a PR cancels the previous run of the same PR. On `master`,
  `verify` and `image` runs queue one by one and are never cancelled.

### Repeated work

With dozens of services, CI time is mostly repeated installs and builds. What
`build.yml` does about it, without dropping a check:

| What | How | Saves |
|---|---|---|
| Docs-only changes | `docs/`, `.claude/` and `*.md` at the root, compared with `HEAD^1` (the PR's base, or the previous squash commit of `master`): install, verify, e2e and the image are skipped; the check passes | a full run, and on `master` a new image and a deploy PR that would change nothing |
| One install and one build for verify and e2e | a web runs `pnpm test:e2e` in the `verify` job on the build `pnpm verify` made, instead of a separate e2e job | per web run: one `pnpm install` and one `next build` |
| Only the containers a service uses | `services` per repo; the minimal starter needs only Postgres | Redis and RabbitMQ startup on every api run that does not use them |
| Dependency layers | the image job writes the layer cache with `mode=max`; PR builds and the next `master` build read it | `pnpm install` inside Docker while the lockfile is unchanged |
| Caches | pnpm store (`pnpm/action-setup`), `.next/cache` for a web | download and recompile time |
| Superseded PR runs | `cancel-in-progress` for `verify` on PRs | runs of commits that were replaced |
| Dependency PRs | Renovate: one weekly PR for all non-major updates per repo, majors apart | dozens of PRs (and runs) a week |

Deliberately not done:

- **Reusing the PR's image on `master`.** A PR's cache and image cannot be read from
  `master` (GitHub cache scopes), and pushing from PR runs would need `packages: write`
  for code that is not merged yet. The `master` build reuses the layer cache instead.
- **Building the image from the `dist`/`.next` of `verify`.** The Dockerfile builds from
  source, so `docker build` gives the same image locally and in CI.
- **Skipping the image build on PRs.** It is the check that the Dockerfile still
  builds; with the layer cache it costs one app build.
- **A matrix or more parallel jobs.** Each job is another runner, checkout and install;
  for these repos one job is cheaper.

## deploy-pr.yml

Job `promote`. Runs only on the default branch.

1. Validates the inputs: digest `sha256:<64 hex>`, image without a tag, full SHA.
2. Gets a token of the App `nick-lukashik-deployer` (`actions/create-github-app-token`):
   `gitops` only, `contents: write`, `pull-requests: write`. It lives one hour and is
   revoked at the end of the job.
3. Checks out `gitops` at the default branch, changes `apps/<service>/prod/kustomization.yaml`,
   runs `kustomize build`: the manifests must contain `<image>@<digest>`.
4. Branch `deploy/<service>-<sha7>` from the current default branch. The commit goes
   through the Contents API as the App: GitHub signs it, no git credentials on the runner.
5. PR to the default branch of `gitops`:
   - title `deploy <service> <sha7>`;
   - body: link to the commit, the image with a link to its version in GHCR, the previous
     digest, link to the build, the commits since the
     previous deploy (up to 50, from `nick-lukashik.com/source-sha`);
   - label `deploy`, reviewer from input `reviewer` (the owner).
6. Open PRs of the same service (branches `deploy/<service>-<sha>`) get the comment
   "Superseded by #N.", are closed, and their branches deleted.
7. A message to the topic "Deploys": service, `sha7`, link to the PR, superseded PRs.

If `gitops` already has this digest, no PR is opened (a notice in the log). A re-run of the
same commit resets its branch and updates its PR; there is never a second PR.

### Digest convention in gitops

`apps/<service>/prod/kustomization.yaml` (the whole service folder is this one directory;
templates `apps/_example-web` and `apps/_example-api`):

```yaml
components:
  - ../../../components/service
  - ../../../components/web # or ../../../components/api
images:
  - name: ghcr.io/nick-lukashik/<service> # the image the components put into the Deployment
    digest: sha256:<64 hex>
commonAnnotations:
  nick-lukashik.com/source-sha: <full SHA of the service commit>
  nick-lukashik.com/source-repo: nick-lukashik/<service repo>
```

- `images` has exactly one entry whose `name` is the image. No `newName`: the
  `kustomize build` check would not find `<image>@<digest>` and the job fails. A
  `newTag` is removed. `make verify` of `gitops` enforces all of this.
- Existing `digest` and `source-sha` values are replaced as text: the comments and blank
  lines of the file stay, the PR changes two lines. A missing key is added by `yq`,
  which reformats the file once. So set up all keys from the start (the templates do).
- `kustomize edit set image` is not used: it reformats the file and drops inline comments.
- `source-sha` drives the commit list of the next PR and the GitHub deployments.
  `commonAnnotations` also land in the pod template.
- The host is not part of this file: `smoke.yml` reads it from the rendered route.

### Inputs

| Input | Default | What it sets |
|---|---|---|
| `image` | required | `needs.ci.outputs.image` |
| `digest` | required | `needs.ci.outputs.digest` |
| `service` | repo name | folder `apps/<service>/` and the name in the PR title |
| `sha` | commit of the run | the service commit being deployed |
| `gitops-repository` | `gitops` | the gitops repo in the same org |
| `gitops-path` | `apps/<service>/prod` | folder with `kustomization.yaml` |
| `reviewer` | `coping-barrel` | who is requested as reviewer; empty for nobody |

Outputs: `pr-url`, `pr-number` (empty when no PR is needed).

## smoke.yml

1. **URL.** Input `url`, or else `https://<host>` of the service's route: `gitops` is
   checked out, `kustomize build <gitops-path>` (default `apps/<service>/prod`) is rendered,
   and the host is annotation `nl-gitops/host` of its only IngressRoute (set by the
   `host` and `subdomain` components). No URL is written anywhere else. `gitops` is
   public, so it is read without a token. The optional `DEPLOYER_APP_ID` and
   `DEPLOYER_PRIVATE_KEY` (a read token of the deployer App) are only for a private
   `gitops`; the examples do not pass them.
2. Every 10 s requests `<url><health-path>` (default `/health/ready`) until it answers `200`
   with header `x-app-version` equal to the expected version. Waits `wait-minutes`
   (10 min): Argo CD polls git every 3 minutes.
3. Every path of `check-paths` (from the route annotation unless explicitly supplied) must answer `200` (redirects followed,
   2 retries).
4. If `package.json` has a `smoke` script, checks out the repo at the deployed commit and
   runs `pnpm smoke` with `SMOKE_URL` and `SMOKE_VERSION` in the env. The script only
   reads: no test accounts in prod, no writes.
5. Result to the topic "Deploys": green in one line; red with the first error, a link to
   the rollback section of the `gitops` deploy runbook and to the log.

On `deployment_status` the job runs only for `state = success` and
`environment = production`. The default version is `deployment.sha`.

### When it runs

- **`deployment_status`** from [`examples/smoke.yml`](examples/smoke.yml). The event appears
  once Argo CD reports deploys to GitHub deployments.
- **By hand** (`workflow_dispatch` of the same file), with or without a version.
- **From `gitops` after every merged deploy PR, until Argo CD reports deployments**: `.github/workflows/smoke.yml`
  of `gitops` (source: [`examples/gitops-smoke.yml`](examples/gitops-smoke.yml)). On a push to
  `master` that changes `apps/*/prod/kustomization.yaml` (a merged deploy or rollback PR) it
  runs `smoke.yml@v1` for every changed `apps/<service>/prod`. It renders each folder and
  passes `url` = `https://` + the `nl-gitops/host` of its IngressRoute; the version and repo
  come from `source-sha` and `source-repo`. Waiting for the version header covers the Argo CD
  sync (Argo CD polls git every 3 minutes, smoke waits `wait-minutes`, 10). Annotation keys
  are matched by suffix, so the file has no domain in it and the domain check of `gitops`
  stays clean. Templates (`apps/_*`) and folders without these annotations (third-party
  apps) are skipped, and so is a folder with no route (the internal APIs, `digests`), with a
  warning. `pnpm smoke` is off there (`gitops` has no access to the packages); the
  HTTP checks remain. Remove it once Argo CD reports deployments, or smoke runs twice.

### Version convention

- In the final stage of the Dockerfile: `ARG APP_VERSION=dev` and `ENV APP_VERSION=$APP_VERSION`.
- The service answers `/health/live` and `/health/ready` with header `x-app-version: <APP_VERSION>`.
- Without the header smoke waits `wait-minutes` and fails. A temporary way out is
  `version-header: ""`: then only the `200` is checked.

### Inputs

| Input | Default | What it sets |
|---|---|---|
| `url` | host of the gitops route | public base URL, `https://…` |
| `gitops-repository` | `gitops` | where the route is read when `url` is empty |
| `gitops-path` | `apps/<service>/prod` | folder of the service in `gitops` |
| `version` | `deployment.sha` or the commit of the run | expected version (SHA) |
| `service` | repo name | name in the message and default folder |
| `repository` | repo of the caller | where `pnpm smoke` comes from |
| `health-path` | `/health/ready` | what is polled |
| `version-header` | `x-app-version` | header with the version; empty to skip the version check |
| `check-paths` | route annotation | space-separated paths that must answer `200`; required with an explicit `url` |
| `wait-minutes` | `10` | how long to wait for the new version |
| `pnpm-smoke` | `true` | run `pnpm smoke` if it exists |
| `github-packages` | `false` | GitHub Packages auth for the `pnpm install` of `pnpm smoke` |
| `rollback-url` | `…/gitops/blob/master/docs/runbook/deploy.md#roll-a-service-back` | rollback link in the red message |

## Secrets per workflow

Passed explicitly by name in the calling job (`secrets:`), never `secrets: inherit`: a
called workflow gets only what it needs.

| Workflow | Secret | Required | What for |
|---|---|---|---|
| `build.yml` | — | | `GITHUB_TOKEN` only (GHCR push, GitHub Packages read) |
| `deploy-pr.yml` | `DEPLOYER_APP_ID` | yes | App token for the PR in `gitops` |
| `deploy-pr.yml` | `DEPLOYER_PRIVATE_KEY` | yes | same |
| `deploy-pr.yml` | `OPS_TELEGRAM_TOKEN` | no | message to "Deploys"; skipped with a warning without it |
| `smoke.yml` | `OPS_TELEGRAM_TOKEN` | no | result to "Deploys"; skipped with a warning without it |
| `smoke.yml` | `DEPLOYER_APP_ID` | no | only for a private `gitops`; not passed while `gitops` is public |
| `smoke.yml` | `DEPLOYER_PRIVATE_KEY` | no | same |

Variables `OPS_TELEGRAM_CHAT_ID` and `OPS_TELEGRAM_TOPIC_DEPLOYS` are read from `vars` by
`deploy-pr.yml` and `smoke.yml`; they are not passed.

## Org secrets and variables

No values here. They are created in **Organization settings → Secrets and variables →
Actions** of the org `nick-lukashik`, access **Selected repositories**.

| Name | Kind | Contents | Read by | Repos with access |
|---|---|---|---|---|
| `DEPLOYER_APP_ID` | secret | App ID of `nick-lukashik-deployer` | `deploy-pr.yml` | service repos |
| `DEPLOYER_PRIVATE_KEY` | secret | App private key (PEM), copy in Bitwarden `deployer-private-key` | `deploy-pr.yml` | service repos |
| `OPS_TELEGRAM_TOKEN` | secret | token of the `ops` bot, Bitwarden `ops-telegram` | `deploy-pr.yml`, `smoke.yml` | service repos, `gitops` (smoke) |
| `OPS_TELEGRAM_CHAT_ID` | variable | supergroup id, Bitwarden `ops-telegram-topics` | `deploy-pr.yml`, `smoke.yml` | same |
| `OPS_TELEGRAM_TOPIC_DEPLOYS` | variable | `message_thread_id` of the topic "Deploys" | `deploy-pr.yml`, `smoke.yml` | same |

- `GITHUB_TOKEN` is created by GitHub for every run; nothing to set up.
- Without `OPS_TELEGRAM_*` the workflows work and skip the message with a warning.
- On GitHub Free, private repos do not see org secrets and variables. `gitops` is public,
  so the org-level names reach it once `gitops` is in their **Selected repositories**.
- `gitops` gets no deployer secrets: nothing there needs to write to it.

## Permissions

| Who | Where | Permissions |
|---|---|---|
| `GITHUB_TOKEN`, job `verify` | service repo | `contents: read`, `packages: read` (only used with `github-packages: true`) |
| `GITHUB_TOKEN`, job `image` | service repo | `contents: read`, `packages: write` (push of this repo's image) |
| `GITHUB_TOKEN`, job `promote` | service repo | `contents: read` (compare commits), `packages: read` (link to the version in GHCR) |
| `GITHUB_TOKEN`, job `smoke` | caller repo | `contents: read`, `packages: read` (only for `pnpm smoke` with `github-packages: true`) |
| GitHub App `nick-lukashik-deployer` | `gitops` only | Contents read/write, Pull requests read/write, Metadata read; no Workflows, no Administration; webhooks off. A 1-hour token per job, in `promote` only |
| Owner `coping-barrel` | `gitops`, service repos | reviewer of deploy PRs; the merge in `gitops` is the deploy |

- **The deployer can open branches and PRs, but cannot change `master` of `gitops`.**
  `gitops` is public, so its `master` ruleset applies on GitHub Free: no push, force push or
  deletion of `master`; changes only through a PR with one approval by someone other than the
  last pusher, and the checks `ci / verify` and `e2e`. The only bypass actor is the organization
  owner; no App or bot is one, and `terraform test` in `infra` fails if one is added
  (`infra` `docs/github.md`). The App cannot approve its own PR. Without the Workflows
  permission it cannot change the workflows of `gitops` either, and without Administration
  it cannot change the rulesets. A leaked deployer key can open a deploy PR; only the
  owner's approval and merge deploys it.
- The bot has no Write on `gitops`: an approval by it would count for the ruleset.
- In PRs from forks `GITHUB_TOKEN` is read-only and org secrets are not passed; images and
  deploy PRs are never created on PRs anyway.
- The calling job grants the maximum (`packages: write` for `ci`); every job inside a
  workflow narrows it to its own.
- Every third-party action is pinned by commit SHA with a version comment; the actionlint
  image by digest; the test service images by exact version.

## If the deploy PR did not open

| What you see | Cause and fix |
|---|---|
| job `deploy` skipped | the image was not pushed: not `master`, `publish` false (template repo), `image` failed, or e2e failed (web) |
| `Create deployer token` failed | `DEPLOYER_*` not given to the repo, the App not installed on `gitops`, or the wrong key |
| `kustomization.yaml not found` | no folder `apps/<service>/prod` in `gitops`, or input `service` does not match it |
| `expected one images entry` | `images` has no entry with the image name, or several |
| `rendered manifests do not use` | the entry has `newName`, or ConfigMap `service` of the folder names another service |
| `kustomize build … failed` | `gitops` does not build right now; fix `master` of `gitops` |
| notice `already has … no PR needed` | `gitops` already has this digest; nothing to deploy |
| 403 on the branch or PR | the App lacks Contents or Pull requests on `gitops` |

Retry: **Re-run failed jobs** in the run of the service. A retry is safe: the branch is
recreated and the existing PR is updated.

## Next

- Not done yet: Renovate in this repo (action SHAs and the test service images of
  `build.yml`, with the `@v1` rule of [Versions](#versions-v1-and-v2)), image scanning and
  signing (Trivy, cosign), a PR title check, a rollback workflow in `gitops` (until then a
  rollback is a revert PR, the red smoke message links to the `gitops` deploy runbook), and
  GitHub deployments reported by Argo CD.
- `actions/create-github-app-token` v3 marks input `app-id` as deprecated in favour of
  `client-id`. Migration: secret `DEPLOYER_CLIENT_ID` and the input in `deploy-pr.yml` and
  `smoke.yml`.
- With stage (`E7-stage`), job `promote-stage` comes before the prod PR: the digest is
  committed to `apps/<service>/stage` without a PR, then e2e; the prod PR opens only if
  e2e is green.

### Per-service smoke paths

The gitops smoke caller reads `nl-gitops/smoke-paths` from the rendered IngressRoute
and passes it as `check-paths` to the reusable workflow. Use space-separated,
read-only paths that return HTTP 200: the web default is `/`, the API default
is `/openapi.json`. Override the annotation in the service folder with a real
functional check. The example API uses `/api/v1/notes?limit=1`; the example web
uses `/ /api/v1/notes?limit=1`, so a degraded readiness response cannot hide a
broken backend. Do not include tokens or other secrets in paths.

When calling `smoke.yml` with an explicit `url`, pass `check-paths` too. Without
`url`, the workflow reads the route annotation automatically. An absent path
list fails configuration instead of silently skipping functional checks.
