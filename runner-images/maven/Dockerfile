# jomietech-runner-maven
#
# Custom GitHub Actions runner image for Maven/Java jobs on the
# jomietech self-hosted runner fleet.
#
# Why this image exists:
#   The stock `maven:3.9-eclipse-temurin-25` image runs as root and
#   targets /root/.m2 by default. When we override --user 1000:1000 (so
#   target/ jars and EFS-mounted ~/.m2 stay ec2-user-owned on the host),
#   Maven blows up trying to write to /root/.m2 unless we also pass
#   `-e HOME=/home/ec2-user -e MAVEN_CONFIG=/home/ec2-user/.m2`. This
#   image bakes in a proper non-root `runner` user with HOME and
#   MAVEN_CONFIG already pointing at the right places, so consumer
#   workflows just run `mvn clean package -DskipTests -B` again.
#
# What this image guarantees:
#   - `runner` user at uid=1000, gid=1000, HOME=/home/runner, shell /bin/bash
#   - /home/runner/.m2 pre-created with runner:runner ownership and 0755
#   - HOME=/home/runner, MAVEN_CONFIG=/home/runner/.m2 baked into image env
#   - Default USER runner, default WORKDIR /home/runner

# Java4Shell (CVE-2026-114938, FORGEERP-62, 2026-11-22): the official
# maven:3.9-eclipse-temurin-25 tag was last rebuilt 2026-09-26 and still ships
# the vulnerable Temurin 25.0.4.1; only the Maven 3.10/4.0 tags moved to the
# fixed 25.0.5.1. To keep Maven 3.9.16 exactly as before, this image now
# rebuilds what the official one does (carlossg/docker-maven) on the fixed
# JDK: Temurin 25.0.5.1+2 on Ubuntu noble, the same apt packages, Maven copied
# from the official 3.9 image, its entrypoint and the mvn symlink.
# Both are pinned by digest; bump them deliberately.
FROM 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/dockerhub/library/maven:3.9-eclipse-temurin-25@sha256:93b8a14ea2f412782e4e842651273b4d903e35cc496284f178fbbe2d67d00976 AS maven39

FROM 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/dockerhub/library/eclipse-temurin:25.0.5.1_2-jdk-noble@sha256:c7eef3dfeb81468533ea48c67bddaa8c7ca916454259e4d0d734d12bcff8e125

RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates curl git openssh-client && \
    rm -rf /var/lib/apt/lists/*
ENV MAVEN_HOME=/usr/share/maven
COPY --from=maven39 /usr/share/maven /usr/share/maven
COPY --from=maven39 /usr/local/bin/mvn-entrypoint.sh /usr/local/bin/mvn-entrypoint.sh
RUN ln -s ${MAVEN_HOME}/bin/mvn /usr/bin/mvn
ENTRYPOINT ["/usr/local/bin/mvn-entrypoint.sh"]
CMD ["mvn"]

# CI tooling baked into the image. `jq` is required by ECS deploy
# workflows (task-definition env/secrets stripping); `curl` is used by
# health-check and webhook steps. Installed up front so consumer
# workflows don't have to apt-get at job time.
RUN apt-get update && \
    apt-get install -y --no-install-recommends jq curl && \
    rm -rf /var/lib/apt/lists/*

# Create the runner user/group at uid 1000 (matches host ec2-user uid on
# the jomietech runner fleet). The maven:3.9-eclipse-temurin-25 base image
# happens to ship a pre-existing group at GID 1000 (no matching named
# user, but the group slot is taken). Defensive teardown of any existing
# uid/gid 1000 occupant lets `useradd --uid 1000 --gid 1000` succeed
# regardless of which base-image revision we're building from.
RUN set -eux; \
    if existing_user=$(getent passwd 1000 | cut -d: -f1) && [ -n "$existing_user" ]; then userdel -r "$existing_user" 2>/dev/null || true; fi; \
    if existing_group=$(getent group 1000 | cut -d: -f1) && [ -n "$existing_group" ]; then groupdel "$existing_group" 2>/dev/null || true; fi; \
    groupadd --gid 1000 runner; \
    useradd --uid 1000 --gid 1000 --create-home --home-dir /home/runner --shell /bin/bash runner; \
    install -d -m 0755 -o runner -g runner /home/runner/.m2

# Fail the image build if the JDK is not the patched one (FORGEERP-62), so a
# digest bump to an unpatched base can never be pushed as :latest.
RUN java -version 2>&1 | tee /dev/stderr | grep -q 'Temurin-25.0.5.1' && mvn -v

# Make HOME and MAVEN_CONFIG correct for the runner user out of the box.
# Consumer workflows can drop their `-e HOME=... -e MAVEN_CONFIG=...`
# overrides; the image-level ENV here is authoritative.
ENV HOME=/home/runner
ENV MAVEN_CONFIG=/home/runner/.m2

# Default exec context.
USER runner
WORKDIR /home/runner
