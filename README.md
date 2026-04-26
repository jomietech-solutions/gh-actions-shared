# gh-actions-shared

Shared GitHub Actions reusable workflows for the `jomietech-solutions` org.

This repo is intentionally **public** so that private repos in the org can call the workflows as `uses:` references. (GitHub's free plan does not allow sharing private reusable workflows across private repos.) The repo holds reusable workflow YAML only — no source code, no infrastructure, no secrets.

## Workflows

### `build-runner-images.yml`

Builds the custom jomietech runner Docker images (`jomietech-runner-node`, `jomietech-runner-maven`, `jomietech-runner-android`) and pushes them to ECR. These images bake in a non-root `runner` user at uid=1000 with HOME=/home/runner and pre-configured tool dirs/PATH so consumer workflows on the self-hosted fleet can run `--user 1000:1000` without per-step env-var patches.

See [`runner-images/README.md`](./runner-images/README.md) for the full design, image guarantees, and consume guide.

Triggers on push to `main` (when `runner-images/**` changes) or manually via `workflow_dispatch`.

### `bootstrap-runner.yml`

Ensures at least one self-hosted runner from the jomietech ASG is online before downstream jobs run. Queue-depth-aware: bursts the ASG when org-wide queued runs exceed a configurable per-runner ratio.

#### Decision matrix

1. No InService/Pending runner -> ASG `desired += 1`
2. Runners exist AND `queued/online > threshold` -> ASG `desired += 1` (burst)
3. Otherwise -> no scale, just wait for current

Always capped at the ASG `MaxSize`.

#### Inputs (all optional, sensible jomietech defaults)

| Input | Default | Purpose |
|---|---|---|
| `asg_name` | `jomietech-fleet-asg` | ASG backing the runner fleet |
| `runner_label` | `jomietech` | Runner label used to count online runners |
| `aws_role` | `arn:aws:iam::171518635415:role/github-actions-bootstrap` | IAM role for OIDC |
| `aws_region` | `ap-southeast-1` | ASG region |
| `threshold` | `3` | Queue/runner ratio above which to scale up |
| `org` | `jomietech-solutions` | Org slug for queue/runner API queries |
| `register_wait_seconds` | `180` | Seconds to wait after first InService for the runner to register |

#### Secrets

| Secret | Required | Purpose |
|---|---|---|
| `runner_api_token` | optional | GitHub PAT with `read:org` + `actions:read`. Without it, threshold autoscaling is disabled and only case 1 (no runner -> +1) fires. |

#### Example consumer (5 lines)

```yaml
jobs:
  bootstrap-runner:
    uses: jomietech-solutions/gh-actions-shared/.github/workflows/bootstrap-runner.yml@main
    secrets:
      runner_api_token: ${{ secrets.RUNNER_API_TOKEN }}

  build:
    needs: bootstrap-runner
    runs-on: [self-hosted, jomietech]
    steps:
      - uses: actions/checkout@v5
      # ...
```

> Note: secrets do **not** auto-inherit across reusable-workflow boundaries. Pass `runner_api_token` explicitly as shown, OR (if `RUNNER_API_TOKEN` becomes an org-level secret on a paid plan) use `secrets: inherit`.

#### Required AWS permissions on the OIDC role

The role must allow:

- `autoscaling:DescribeAutoScalingGroups`
- `autoscaling:SetDesiredCapacity` (scoped to the target ASG)

#### Pinning

For reproducibility, pin to a tag or commit SHA in production workflows:

```yaml
uses: jomietech-solutions/gh-actions-shared/.github/workflows/bootstrap-runner.yml@v1
```

`@main` follows the latest changes — fine for internal use, riskier for external consumers.

## License

MIT — see [LICENSE](./LICENSE).
