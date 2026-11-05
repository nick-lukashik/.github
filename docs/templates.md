# Starters, examples and the services made from them

Two kinds of reference repos, per stack:

| Repo | Role | CI |
|---|---|---|
| `nest-starter`, `next-starter` | templates: the base every service needs (HTTP, config, logs, health, metrics; Postgres for an api; SEO, CSP, i18n for a web) and one sample module or page | verify; publishes only in repos created from it |
| `nest-example`, `next-example` | not templates: every capability wired and tested together (Redis, RabbitMQ outbox/inbox, pg-boss; the `/api` proxy and TanStack Query) | verify only (`publish: false`), never deployed |

A service is created from a starter and gets only the capabilities it uses, copied from
the example ("Add a capability" in the nest-starter README, "Add the API" in the
next-starter README). The shared code (`src/common/` of the nest repos; `src/proxy.ts`,
`src/lib/{env,security,seo}/`, `src/i18n/` of the next repos) is identical in a starter and
its example, so a capability is a folder plus its wiring. The exceptions in the next repos,
each with its test where it has one: `src/lib/env/server.ts` (the example adds `API_URL` and
its budgets), `src/lib/security/csp.ts` (the example allows the styles of its toasts) and
`src/i18n/messages/*` (each repo's own text). A fix to shared code is made in the example
first, then copied to the starter.

From then on a service is a copy: a fix in a starter or an example does not reach it by
itself. This page is how fixes get there until the shared code is a package.

## Record where a service came from

A service records its starter in `.template-version` in its root, created together with
the repo. So far only `landing-web` has one: `id-web`, `id-api`, `notifications-api`,
`wallet-api`, `pay-api` and `digests` were created without it, so nothing records how far
behind they are. The file:

```
repo=nick-lukashik/nest-starter
commit=<full SHA of the starter commit it was created from or last synced to>
```

Get the commit right after creating the repo:

```sh
printf 'repo=nick-lukashik/%s\ncommit=%s\n' nest-starter \
  "$(gh api repos/nick-lukashik/nest-starter/commits/master --jq .sha)" > .template-version
```

`commit` moves forward every time fixes are ported from the starter, in the same PR.

## Port a fix

1. See what changed in the starter since the recorded commit:

   ```sh
   . ./.template-version
   git remote add starter "https://github.com/$repo.git" 2>/dev/null || true
   git fetch starter master
   git log --oneline "$commit..starter/master"
   git diff "$commit" starter/master -- . ':!README.md' ':!CHANGELOG.md'
   ```

2. Apply what applies to this service: `git cherry-pick` a starter commit when the files
   still match, otherwise apply the diff by hand (`git diff ... | git apply -3`).
   A fix to a capability the service copied from the example (messaging, Redis, jobs,
   the API proxy) is taken from the example the same way.
3. Set `commit` in `.template-version` to the starter commit you synced to, `pnpm verify`,
   one PR titled `chore: sync <starter> to <sha7>`.

A fix that matters for security (auth, headers, CSP, input limits, dependency CVEs) or for
messaging (outbox, retries, dead letters) names the affected services in its PR, and each
gets its sync PR. The owner checks `.template-version` of every service that has one at each
stage review: a service far behind its starter is a finding.

## Packages: the plan

Manual porting does not scale to dozens of services. The shared code did not move with the
first real service (auth): the `packages` repo (GitHub Packages) holds only
`@nick-lukashik/auth` and `@nick-lukashik/contracts`, and `id-api`, `notifications-api`,
`wallet-api` and `pay-api` still carry their own `src/common/`. When it moves, it goes into
small versioned packages `@nick-lukashik/<name>` in that repo, and services get fixes as a
Renovate update:

- one package per independently usable capability, for example HTTP and logging
  (errors, request id, validation, health, metrics), database (pool, time budgets,
  migrations), messaging (RabbitMQ, outbox, inbox); not one package that pulls in Redis,
  RabbitMQ and pg-boss for everyone, and not a package per function;
- business modules and each service's schema stay in the service;
- the starters and examples then depend on the packages instead of carrying `src/common/`;
- such services call `build.yml` with `github-packages: true`, as the account APIs do
  for `auth` and `contracts` ([ci.md](ci.md)).
