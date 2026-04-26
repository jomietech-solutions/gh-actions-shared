# jomietech-runner-node
#
# Custom GitHub Actions runner image for Node/Yarn/npm jobs on the
# jomietech self-hosted runner fleet.
#
# Why this image exists:
#   The stock `node:20-bookworm` image runs as root by default and writes
#   to /root/.cache, /usr/local/bin, etc. When we override with
#   --user 1000:1000 (to keep workspace files ec2-user-owned on the host
#   when bind-mounted via EFS), every tool — corepack, yarn, npm, pnpm —
#   needs its own env-var override (HOME, COREPACK_HOME, NPM_CONFIG_CACHE,
#   YARN_CACHE_FOLDER, PATH adjust for ~/.local/bin shims) or it fails
#   with EACCES on /usr/local/bin or /root/.cache. We've patched these
#   piecemeal across consumer workflows; the next new tool would hit the
#   same wall. This image bakes in a proper non-root user, a pre-enabled
#   corepack/yarn shim under that user's HOME, and the right PATH — so
#   consumer workflows just run `yarn install && yarn build` again.
#
# What this image guarantees:
#   - `runner` user at uid=1000, gid=1000, HOME=/home/runner, shell /bin/bash
#   - /home/runner/{.cache/yarn, .cache/node/corepack, .npm, .local/bin}
#     pre-created with runner:runner ownership and 0755
#   - corepack pre-enabled into /home/runner/.local/bin (uid-1000-writable)
#   - yarn classic 1.22.22 pre-prepared via corepack
#   - /home/runner/.local/bin prepended to PATH for all shells (login + non-login)
#   - Default USER runner, default WORKDIR /home/runner

FROM 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/dockerhub/library/node:20-bookworm

# Pinned yarn classic version. Matches what every consumer repo uses and
# what `corepack prepare ... --activate` will install for the runner user.
ARG YARN_VERSION=1.22.22

# Create the runner user/group at uid 1000 (matches host ec2-user uid on
# the jomietech runner fleet, so EFS bind-mounts have consistent ownership).
# The base node image already creates a `node` user at uid 1000; we delete
# it first to free up the uid for `runner`. usermod can't be used because
# we want the literal name `runner` and the literal HOME path /home/runner.
RUN set -eux; \
    if existing_user=$(getent passwd 1000 | cut -d: -f1) && [ -n "$existing_user" ]; then userdel -r "$existing_user" 2>/dev/null || true; fi; \
    if existing_group=$(getent group 1000 | cut -d: -f1) && [ -n "$existing_group" ]; then groupdel "$existing_group" 2>/dev/null || true; fi; \
    groupadd --gid 1000 runner; \
    useradd --uid 1000 --gid 1000 --create-home --home-dir /home/runner --shell /bin/bash runner; \
    install -d -m 0755 -o runner -g runner \
        /home/runner/.cache \
        /home/runner/.cache/yarn \
        /home/runner/.cache/node \
        /home/runner/.cache/node/corepack \
        /home/runner/.npm \
        /home/runner/.local \
        /home/runner/.local/bin

# Make /home/runner/.local/bin available to every shell that sources
# /etc/profile (login shells AND `bash -lc` invocations from CI). Setting
# this at /etc/profile.d ensures even non-login interactive shells see it
# when /etc/profile is loaded. CI steps that invoke `bash -c '...'` get
# the PATH from the ENV directive below.
RUN echo 'export PATH=/home/runner/.local/bin:$PATH' > /etc/profile.d/runner.sh && \
    chmod 0644 /etc/profile.d/runner.sh

# Prepend /home/runner/.local/bin to the image PATH so non-shell exec
# (docker run ... <cmd>, github actions step `run:` blocks under sh -c)
# also picks up the corepack-managed yarn shim without sourcing /etc/profile.
ENV PATH=/home/runner/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV HOME=/home/runner
ENV YARN_CACHE_FOLDER=/home/runner/.cache/yarn
ENV NPM_CONFIG_CACHE=/home/runner/.npm
ENV COREPACK_HOME=/home/runner/.cache/node/corepack

# Pre-enable corepack into runner's writable bin dir (uid-1000-writable)
# and pre-prepare yarn 1.22.22 so the first CI step doesn't have to. Doing
# this AS THE RUNNER USER ensures every file written ends up uid 1000 owned.
USER runner
WORKDIR /home/runner

RUN corepack enable --install-directory /home/runner/.local/bin && \
    corepack prepare yarn@${YARN_VERSION} --activate && \
    /home/runner/.local/bin/yarn --version

# Image default exec context. Consumer workflows can still pass
# --user 1000:1000 explicitly — it's a no-op against this image but keeps
# the pattern consistent with other (legacy) jobs that need it.
USER runner
WORKDIR /home/runner
