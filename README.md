# gh-actions-shared

> Shared, reusable GitHub Actions workflows for the `jomietech-solutions` org.

This repo holds the workflow YAML and runner-image Dockerfiles that other org repos call via `uses:`. It is intentionally **public** so private repos in the org can reference its workflows — GitHub's free plan does not allow sharing reusable workflows across private repos. No application source, no infrastructure, no secrets live here.

## Contents

### Reusable workflows (`.github/workflows/*.yml`)

| Workflow | Purpose | Trigger |
|---|---|---|
| [`bootstrap-runner.yml`](./.github/workflows/bootstrap-runner.yml) | Ensures at least one self-hosted runner from the jomietech ASG is online (and bursts capacity when org-wide queue depth exceeds a per-runner threshold) before downstream jobs run. Capped at the ASG `MaxSize`. | `workflow_call` |
| [`build-runner-images.yml`](./.github/workflows/build-runner-images.yml) | Builds and pushes the three custom runner images (`jomietech-runner-node`, `jomietech-runner-maven`, `jomietech-runner-android`) to ECR. Uses `bootstrap-runner.yml` itself, then runs three parallel build jobs on `[self-hosted, jomietech]`, authenticating to ECR via the runner instance role. | `workflow_dispatch` + `push` to `main` when `runner-images/**` or the workflow itself changes |

### Runner images (`runner-images/`)

Dockerfiles wrapped around upstream language tool images, baking a non-root `runner` user (uid=1000) with `HOME=/home/runner` and pre-configured tool cache dirs/PATH so consumer workflows can run `--user 1000:1000` without per-step env-var patches.

- `runner-images/node/` — Node.js builds (Next.js, Vite, Yarn, npm, EAS CLI)
- `runner-images/maven/` — Maven + Java builds (Spring Boot services)
- `runner-images/android/` — Android SDK + Gradle (mobile EAS builds)

See [`runner-images/README.md`](./runner-images/README.md) for image guarantees and the consume guide.

## Usage

Reference workflows from a calling repo via `uses: jomietech-solutions/gh-actions-shared/.github/workflows/<name>.yml@<ref>`.

### `bootstrap-runner.yml` — example consumer

```yaml
jobs:
  bootstrap-runner:
    uses: jomietech-solutions/gh-actions-shared/.github/workflows/bootstrap-runner.yml@main
    secrets:
      runner_api_token: ${{ secrets.RUNNER_API_TOKEN }}

  build:
    needs: bootstrap-runner
    runs-on: [self-hosted, jomietech]
    container: 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-node:latest
    steps:
      - uses: actions/checkout@v5
      # ...
```

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

> Reusable-workflow secrets do **not** auto-inherit. Pass `runner_api_token` explicitly as shown, OR use `secrets: inherit` once `RUNNER_API_TOKEN` is an org-level secret on a paid plan.

#### Required AWS permissions on the OIDC role

- `autoscaling:DescribeAutoScalingGroups`
- `autoscaling:SetDesiredCapacity` (scoped to the target ASG)

### `build-runner-images.yml`

Triggered on push to `main` under `runner-images/**` or via the Actions tab (`workflow_dispatch`). No consumer-side wiring — the workflow lives and runs in this repo.

### Pinning

For reproducibility, pin to a tag or commit SHA in production workflows:

```yaml
uses: jomietech-solutions/gh-actions-shared/.github/workflows/bootstrap-runner.yml@v1
```

`@main` follows latest changes — fine for internal use, riskier for downstream pinning.

## Conventions

- **Self-hosted jomietech runners only.** All consumer jobs target `runs-on: [self-hosted, jomietech]`. Do not use `ubuntu-latest` for org workloads — the only `ubuntu-latest` job in this repo is the bootstrap step itself, which has to run *before* a self-hosted runner exists. (See memory `feedback_runner_jomietech.md`.)
- **No installs on the runner host.** Consumer workflows must use the `container:` directive with one of the prebuilt images (`jomietech-runner-node` / `-maven` / `-android` from `171518635415.dkr.ecr.ap-southeast-1.amazonaws.com`). Never `apt-get install` / `npm install -g` / `pip install` directly on the runner host — it pollutes the shared fleet and breaks subsequent jobs. (See memory `feedback_no_runner_host_install.md`.)
- **OIDC, not long-lived keys.** Consumers needing AWS auth should use `aws-actions/configure-aws-credentials@v4` with `role-to-assume`. The build jobs in this repo skip `role-to-assume` because they run on self-hosted runners that already have an EC2 instance role with ECR push.
- **`@main` is the moving ref.** Internal callers may pin to `@main`; external/critical pipelines should pin to a tag or commit SHA.
- **No long-running pre-push hooks** in consumer repos — SSH connections die during 7–8 min hooks and the push silently fails. (See memory `feedback_no_long_prepush_hooks.md`.)

## Related

Repos in the `jomietech-solutions` org that consume these workflows:

- [`forgeerp-api-tests`](https://github.com/jomietech-solutions/forgeerp-api-tests) — API regression + smoke suites; both `api-regression.yml` and `api-smoke.yml` call `bootstrap-runner.yml@main` before the test job.
- [`terraform-jomietech-base`](https://github.com/jomietech-solutions/terraform-jomietech-base) — provisions the underlying runner ASG (`130-Github-Runner.tf`) and the ECR repos that hold the runner images built here.
- ForgeERP product repos (`enterprise-resource-planning`, `enterprise-resource-planning-backend`, `backend-common`, `frontend-common`, `forgeerp-mobile`) — deploy via self-hosted runners; their workflows are expected to migrate to `bootstrap-runner.yml` as they're touched.

## License

MIT — see [LICENSE](./LICENSE).
