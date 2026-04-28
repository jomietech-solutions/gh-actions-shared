# jomietech-runner-android
#
# Custom GitHub Actions runner image for React Native Android (Gradle)
# jobs on the jomietech self-hosted runner fleet.
#
# Why this image exists:
#   The stock `reactnativecommunity/react-native-android:v20.1` image
#   runs as root with no defined non-root user. When we override
#   --user 1000:1000 (so EFS-mounted /home/ec2-user/.gradle stays
#   ec2-user-owned on the host), Gradle blows up because uid 1000 has no
#   writable HOME, no entry in /etc/passwd, and Gradle's transform/cache
#   directories under HOME aren't pre-created. We've worked around this
#   by passing `-e GRADLE_USER_HOME=...` and chmod 0777 on /opt/android.
#   This image bakes in a proper `runner` user at uid 1000 with the right
#   HOME and pre-created Gradle dirs so consumer workflows just run
#   `./gradlew assembleDebug --no-daemon` again.
#
# What this image guarantees:
#   - `runner` user at uid=1000, gid=1000, HOME=/home/runner, shell /bin/bash
#   - /home/runner/{.gradle, .cache/yarn, .npm, .local/bin} pre-created
#     with runner:runner ownership and 0755
#   - corepack pre-enabled + yarn 1.22.22 ready (needed for `npx expo prebuild`)
#   - /opt/android remains 0777 (verified — already set by upstream image)
#   - HOME=/home/runner, GRADLE_USER_HOME=/home/runner/.gradle baked in
#   - YARN_CACHE_FOLDER, NPM_CONFIG_CACHE point at runner-writable dirs
#   - Default USER runner, default WORKDIR /home/runner

FROM 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/dockerhub/reactnativecommunity/react-native-android:v20.1

# CI tooling baked into the image. `jq` is required by ECS deploy
# workflows (task-definition env/secrets stripping); `curl` is used by
# health-check and webhook steps. Installed up front so consumer
# workflows don't have to apt-get at job time.
RUN apt-get update && \
    apt-get install -y --no-install-recommends jq curl && \
    rm -rf /var/lib/apt/lists/*

# Pinned yarn classic version (matches forgeerp-mobile's package manager).
ARG YARN_VERSION=1.22.22

# Create the runner user/group at uid 1000 (matches host ec2-user uid on
# the jomietech runner fleet). The upstream RN image runs as root with no
# named non-root user, so creating `runner` cleanly takes the slot.
#
# Verify /opt/android is world-writable (the upstream image sets
# `chmod -R 0777 /opt/android` per its Dockerfile, but we re-apply it
# defensively in case a base layer change removes it). Without this,
# `npx expo prebuild` and Gradle SDK manager writes will EACCES under
# uid 1000.
RUN set -eux; \
    if existing_user=$(getent passwd 1000 | cut -d: -f1) && [ -n "$existing_user" ]; then userdel -r "$existing_user" 2>/dev/null || true; fi; \
    if existing_group=$(getent group 1000 | cut -d: -f1) && [ -n "$existing_group" ]; then groupdel "$existing_group" 2>/dev/null || true; fi; \
    groupadd --gid 1000 runner; \
    useradd --uid 1000 --gid 1000 --create-home --home-dir /home/runner --shell /bin/bash runner; \
    install -d -m 0755 -o runner -g runner \
        /home/runner/.gradle \
        /home/runner/.cache \
        /home/runner/.cache/yarn \
        /home/runner/.cache/node \
        /home/runner/.cache/node/corepack \
        /home/runner/.npm \
        /home/runner/.local \
        /home/runner/.local/bin; \
    chmod -R 0777 /opt/android

# Same /etc/profile.d trick as the node image — ensures `bash -lc '...'`
# CI steps see the runner-owned bin dir on PATH.
RUN echo 'export PATH=/home/runner/.local/bin:$PATH' > /etc/profile.d/runner.sh && \
    chmod 0644 /etc/profile.d/runner.sh

# Image-level env so non-shell exec (docker run ... <cmd>) and github
# actions `run:` blocks under sh -c also pick up the right PATH/HOME.
ENV PATH=/home/runner/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV HOME=/home/runner
ENV GRADLE_USER_HOME=/home/runner/.gradle
ENV YARN_CACHE_FOLDER=/home/runner/.cache/yarn
ENV NPM_CONFIG_CACHE=/home/runner/.npm
ENV COREPACK_HOME=/home/runner/.cache/node/corepack

# Pre-enable corepack and yarn for the runner user. `npx expo prebuild`
# and any yarn-driven RN tooling will work first-call without scaffolding
# a yarn shim at runtime.
USER runner
WORKDIR /home/runner

RUN corepack enable --install-directory /home/runner/.local/bin && \
    corepack prepare yarn@${YARN_VERSION} --activate && \
    /home/runner/.local/bin/yarn --version

USER runner
WORKDIR /home/runner
