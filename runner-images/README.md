# jomietech runner images

Custom Docker images that wrap the upstream language tool images with a
pre-configured non-root `runner` user (uid=1000, gid=1000,
HOME=/home/runner). Used by every CI job on the jomietech self-hosted
runner fleet so that workspace files written via EFS bind mounts stay
`ec2-user`-owned on the host (matching the host's uid 1000), without
each consumer workflow having to patch HOME, COREPACK_HOME,
NPM_CONFIG_CACHE, YARN_CACHE_FOLDER, MAVEN_CONFIG, GRADLE_USER_HOME,
or PATH every time we add a new tool.

## Images

| ECR repo | Base image | Used by |
|---|---|---|
| `jomietech-runner-node` | `dockerhub/library/node:20-bookworm` | yarn/npm/corepack — frontend builds + mobile lint-test/EAS submission |
| `jomietech-runner-maven` | `dockerhub/library/maven:3.9-eclipse-temurin-25` | mvn — backend builds (auth, ERP) |
| `jomietech-runner-android` | `dockerhub/reactnativecommunity/react-native-android:v20.1` | gradle assembleDebug — mobile Android local APK |

All three images:
- Create a `runner` user at uid=1000, gid=1000, HOME=`/home/runner`, shell `/bin/bash`
- Pre-create the tool cache dirs under `/home/runner` with `runner:runner` ownership (0755)
- Bake `HOME=/home/runner` (and tool-specific env: `MAVEN_CONFIG`, `GRADLE_USER_HOME`, `YARN_CACHE_FOLDER`, `NPM_CONFIG_CACHE`, `COREPACK_HOME`) into the image env
- Set `USER runner` and `WORKDIR /home/runner` as the default exec context

The node + android images additionally pre-enable corepack and pre-prepare yarn 1.22.22 into `/home/runner/.local/bin` (which is on PATH).

## Pull URLs

```
171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-node:latest
171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-maven:latest
171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-android:latest
```

`:<short-sha>` tags are also pushed for every build so a workflow can pin to a known image SHA if needed.

## How to consume

### Container-style job

```yaml
test:
  needs: bootstrap-runner
  runs-on: [self-hosted, jomietech]
  container:
    image: 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-maven:latest
    options: --user 1000:1000     # explicit, but a no-op vs the baked-in runner user
    volumes:
      - /home/ec2-user/.m2:/home/runner/.m2  # in-container path now /home/runner/...
  steps:
    - uses: actions/checkout@v6
    - run: mvn clean package -DskipTests -B
```

### `docker run` step

```yaml
- name: Build JAR
  run: |
    docker run --rm \
      --user 1000:1000 \
      -v /home/ec2-user/.m2:/home/runner/.m2 \
      -v "$PWD":/build -w /build \
      171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/jomietech-runner-maven:latest \
      mvn clean package -DskipTests -B
```

Note the EFS bind-mount target is now `/home/runner/...` (matching the new HOME) instead of `/home/ec2-user/...`. The host source path is unchanged.

## When to rebuild

Trigger the **Build runner images** workflow (`gh workflow run build-runner-images.yml -R jomietech-solutions/gh-actions-shared`) when:

1. The upstream base image changes (e.g. node 20 -> node 22, maven 3.9 -> 4.0, RN community image v20.x -> v21.x). Update the `FROM` line in the relevant Dockerfile and push to `main` — the workflow's path filter will auto-trigger.
2. We add a new pre-installed tool (e.g. add `pnpm` to the node image, add `kotlin` to the android image).
3. The pinned yarn version (`YARN_VERSION` ARG) needs to bump.
4. A security scan flags a CVE in the base image — rebuild against the latest base tag with `workflow_dispatch`.

The `latest` tag rolls forward on every successful build; the `:<short-sha>` tag is immutable per build.

## Build workflow

Lives at `.github/workflows/build-runner-images.yml`. It:
- Calls the org-level `bootstrap-runner.yml` to ensure a self-hosted runner is online
- Runs three parallel build jobs (`build-node`, `build-maven`, `build-android`) on `[self-hosted, jomietech]`
- Each job: checkout -> ECR login -> docker buildx build/push to `latest` + `<short-sha>`
- Pushes go to ECR via the runner's instance role (no extra IAM needed). Repos are auto-created on first push (the runner role has `ecr:CreateRepository`).
- Triggers: `workflow_dispatch` (manual) and `push` to `main` when `runner-images/**` changes.
