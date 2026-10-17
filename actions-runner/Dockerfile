# Pin these base images to tested versions/digests in production.
FROM moby/buildkit:rootless AS buildkit

FROM ghcr.io/actions/actions-runner:latest

USER root

# Copy the BuildKit client only; the daemon runs in a sidecar.
COPY --from=buildkit /usr/bin/buildctl /usr/local/bin/buildctl

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       ca-certificates \
       curl \
       gnupg \
       clamav \
       clamav-freshclam \
    && rm -rf /var/lib/apt/lists/*

# Install a pinned Trivy release from its official repository.
ARG TRIVY_VERSION=0.69.3
RUN curl -fsSL \
      "https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.deb" \
      -o /tmp/trivy.deb \
    && apt-get update \
    && apt-get install -y /tmp/trivy.deb \
    && rm -f /tmp/trivy.deb \
    && rm -rf /var/lib/apt/lists/*

USER runner

ENV BUILDKIT_HOST=tcp://127.0.0.1:1234

WORKDIR /home/runner 