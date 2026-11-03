# .github

Organization-wide files of `nick-lukashik`: the reusable GitHub Actions workflows, the service runtime contract they implement, and the [organization profile](profile/README.md).

| Path | What |
|---|---|
| [`.github/workflows/build.yml`](.github/workflows/build.yml) | verify, Playwright e2e for a web, the image to GHCR by digest |
| [`.github/workflows/deploy-pr.yml`](.github/workflows/deploy-pr.yml) | the deploy PR in `gitops` with the new digest; the owner's merge is the deploy |
| [`.github/workflows/smoke.yml`](.github/workflows/smoke.yml) | smoke check after a deploy, result to Telegram |
| [`docs/service-contract.md`](docs/service-contract.md) | what every service image provides and what `gitops` and CI rely on |
| [`docs/ci.md`](docs/ci.md) | how to connect a service: inputs, secrets, permissions, versions |
| [`docs/templates.md`](docs/templates.md) | starters and examples, and how fixes reach the services made from them |
| [`docs/examples/`](docs/examples) | `ci.yml` of a web and an api, the smoke callers |
| [`profile/README.md`](profile/README.md) | the organization profile |

Callers pin the moving major tag `@v1`, never `@master`; how `v1` moves: [docs/ci.md](docs/ci.md#versions-v1-and-v2).

The call in a service repo, `.github/workflows/ci.yml` (full versions: [ci-web.yml](docs/examples/ci-web.yml), whose `verify` also runs the Playwright e2e, and [ci-api.yml](docs/examples/ci-api.yml) with the test services):

```yaml
jobs:
  ci:
    uses: nick-lukashik/.github/.github/workflows/build.yml@v1
    permissions: { contents: read, packages: write }
    with:
      kind: api # or web
      services: postgres # only what the service uses: postgres, redis, rabbitmq
      publish: ${{ !github.event.repository.is_template }}
  deploy:
    needs: ci
    if: needs.ci.outputs.digest != ''
    uses: nick-lukashik/.github/.github/workflows/deploy-pr.yml@v1
    permissions: { contents: read, packages: read }
    with: { image: "${{ needs.ci.outputs.image }}", digest: "${{ needs.ci.outputs.digest }}" }
    secrets:
      DEPLOYER_APP_ID: ${{ secrets.DEPLOYER_APP_ID }}
      DEPLOYER_PRIVATE_KEY: ${{ secrets.DEPLOYER_PRIVATE_KEY }}
      OPS_TELEGRAM_TOKEN: ${{ secrets.OPS_TELEGRAM_TOKEN }}
```

Check:

```bash
docker run --rm -v "$PWD:/repo" -w /repo rhysd/actionlint:1.7.12 -color .github/workflows/*.yml docs/examples/*.yml
```
