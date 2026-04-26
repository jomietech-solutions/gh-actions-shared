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

FROM 171518635415.dkr.ecr.ap-southeast-1.amazonaws.com/dockerhub/library/maven:3.9-eclipse-temurin-25

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

# Make HOME and MAVEN_CONFIG correct for the runner user out of the box.
# Consumer workflows can drop their `-e HOME=... -e MAVEN_CONFIG=...`
# overrides; the image-level ENV here is authoritative.
ENV HOME=/home/runner
ENV MAVEN_CONFIG=/home/runner/.m2

# Default exec context.
USER runner
WORKDIR /home/runner
