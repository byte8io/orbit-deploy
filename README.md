# orbit-deploy

GitHub Action that ships a Magento build to [Orbit](https://orbit.byte8.io)
in one job step. Wraps `orbit-agent build-and-upload` — see the
[Orbit docs](https://docs.byte8.io/orbit/) for the underlying flow.

## Quick start

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - name: Build the artifact
        run: |
          composer install --no-dev --optimize-autoloader
          bin/magento setup:di:compile
          bin/magento setup:static-content:deploy -f en_US
      - name: Ship to Orbit
        uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_ENV_ID }}
          source-dir: '.'
          exclude: |
            .git
            .github
            tests
            *.log
          pat: ${{ secrets.ORBIT_PAT }}
```

That's the whole CI surface. The job blocks until the deploy reaches a
terminal state and exits with the matching code, so a green build means
the new release is live.

## Setup (one-time)

1. **Mint a Personal Access Token** in the Orbit dashboard → Settings →
   Personal Access Tokens. Copy the `pat_…` value (shown once).
2. **Find your environment UUID** in the Orbit dashboard → Environments
   → click the env → the URL ends in the UUID.
3. **Store both in your GitHub repo**:
   - `Settings → Secrets and variables → Actions → Secrets`: add
     `ORBIT_PAT` with the PAT value.
   - `Settings → Secrets and variables → Actions → Variables`: add
     `ORBIT_ENV_ID` with the UUID (non-secret — env UUIDs are not
     credentials).
4. **Drop the `Ship to Orbit` step into your workflow** (see Quick start
   above).

## Inputs

| Name | Required | Default | Description |
|---|---|---|---|
| `env-id` | ✓ | — | Orbit environment UUID. |
| `pat` | ✓ | — | Personal Access Token (`pat_…`). Pass via `${{ secrets.ORBIT_PAT }}`. |
| `source-dir` | one of | — | Directory to tar up and ship. Mutually exclusive with `archive`. |
| `archive` | one of | — | Path to a pre-built `.tar.gz`. Mutually exclusive with `source-dir`. |
| `exclude` |  | — | Newline-separated tar exclude patterns (`.git`, `node_modules`, `*.log`). Only used with `source-dir`. |
| `git-ref` |  | `${{ github.sha }}` | Git ref recorded on the deployment row + REVISION file. |
| `version` |  | auto | Release directory name override. Auto-generated server-side when omitted. |
| `deploy-type` |  | `full` | `full` or `code`. |
| `watch` |  | `true` | Block until terminal status. Job exit code matches the deploy. |
| `watch-timeout-secs` |  | `1800` | Hard cap on `--watch` (deploy keeps running on the agent regardless). |
| `no-deploy` |  | `false` | Upload only — skip `createDeployment`. Useful for pre-staging. |
| `api-url` |  | `https://orbit.byte8.io/graphql` | Override for self-hosted Orbit. |
| `agent-version` |  | `latest` | Pin to a specific `orbit-agent` tag for reproducible CI. |

## Patterns

### Ship a pre-built artifact from an earlier job

Useful when your CI already produces a `release.tar.gz` and you don't
want this action to repackage it:

```yaml
      - name: Ship to Orbit
        uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_ENV_ID }}
          archive: ./release.tar.gz
          pat: ${{ secrets.ORBIT_PAT }}
```

### Promote staging → production via two jobs

```yaml
jobs:
  staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: composer install --no-dev && bin/magento setup:di:compile
      - uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_STAGING_ENV_ID }}
          source-dir: .
          pat: ${{ secrets.ORBIT_PAT }}

  prod:
    needs: staging
    runs-on: ubuntu-latest
    environment: production    # requires reviewer approval
    steps:
      - uses: actions/checkout@v4
      - run: composer install --no-dev && bin/magento setup:di:compile
      - uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_PROD_ENV_ID }}
          source-dir: .
          pat: ${{ secrets.ORBIT_PAT }}
```

Alternatively, build once on staging and use the Orbit dashboard's
**Promote** button to ship the same artifact bytes to prod — no second
build needed.

### Fire-and-forget (don't block on the deploy)

```yaml
      - uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_ENV_ID }}
          source-dir: .
          watch: 'false'
          pat: ${{ secrets.ORBIT_PAT }}
```

The job returns as soon as the artifact is uploaded and the deployment
is queued. The Orbit agent picks it up on its next poll.

### Pre-stage without deploying

```yaml
      - uses: byte8io/orbit-deploy@v1
        with:
          env-id: ${{ vars.ORBIT_ENV_ID }}
          source-dir: .
          no-deploy: 'true'
          pat: ${{ secrets.ORBIT_PAT }}
```

Uploads the artifact to R2 but doesn't create a deployment. Use when a
human will trigger the actual deploy from the dashboard.

## Notifications

The action's only output is the job's exit code (success / failure).
For Slack / Discord / Mattermost notifications on every deploy outcome,
configure the **Notification Webhook URL** on the env in the Orbit
dashboard → Operations tab. Fires from orbit-server on terminal status,
independent of the CI run.

## Troubleshooting

**"Input 'pat' must be a Personal Access Token starting with pat_"** —
You probably pasted an agent token (`obt_…`). Agent tokens authenticate
the host-side polling agent against Orbit's internal API; this Action
uses the public GraphQL endpoint and needs a PAT instead. Mint one in
dashboard Settings.

**"A deployment is already in progress on this environment"** —
Another deploy (CI workflow, dashboard click, or a stuck row) is
holding the env's deploy:lock. Wait for it to finish or use the
**Cancel** button on the running deployment's detail page if it's
stuck. See the [deploy:lock docs](https://docs.byte8.io/orbit/deploy-lock).

**"orbit-agent not found after install"** — The install script
couldn't drop the binary into `~/.local/bin`. Check the install log
above the failure for the underlying error. If you're on a non-Linux
runner, verify `orbit-agent` ships a binary for that platform.

**"PAT bearer present but invalid / expired / revoked"** — The PAT was
revoked or expired. Mint a new one and update the `ORBIT_PAT` secret.

**Deploy timed out via `--watch`** — The Orbit agent on the host is
still running the deploy; only the CLI gave up waiting. Check the
dashboard for live progress. Bump `watch-timeout-secs` if your typical
deploys exceed 30 minutes.

## License

MIT — see [LICENSE](./LICENSE).
