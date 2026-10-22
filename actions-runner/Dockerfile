FROM ubuntu:latest AS clamav

ARG CLAMAV_DEB_URL=https://www.clamav.net/downloads/production/
ARG CLAMAV_DEB_FILE=clamav-1.5.4.linux.x86_64.deb
ARG CLAMAV_SIG_URL=https://www.clamav.net/downloads/production/
ARG CLAMAV_SIG_FILE=clamav-1.5.4.linux.x86_64.deb.sig
ARG CLAMAV_GPG_URL=https://raw.githubusercontent.com/Cisco-Talos/clamav-documentation/main/src/manual/
ARG CLAMAV_GPG_FILE=cisco-talos.gpg
ARG EXPECTED_FINGERPRINT="5BADCA2665EF59DCF8A23D8B707F0DB480836771"

WORKDIR /tmp

RUN apt update \
    && apt install -y --no-install-recommends \
       ca-certificates \
       wget \
       gnupg

RUN set -eu; \
    DEB="${CLAMAV_DEB_URL}${CLAMAV_DEB_FILE}"; \
    SIGNATURE="${CLAMAV_SIG_URL}${CLAMAV_SIG_FILE}"; \
    GPG="${CLAMAV_GPG_URL}${CLAMAV_GPG_FILE}"; \
    wget -O "${CLAMAV_DEB_FILE}" "$DEB"; \
    wget -O "${CLAMAV_SIG_FILE}" "$SIGNATURE"; \
    wget -O "${CLAMAV_GPG_FILE}" "$GPG"; \ 
    ACTUAL_FINGERPRINT="$(gpg --show-keys --with-colons "${CLAMAV_GPG_FILE}" \
        | awk -F: '$1 == "fpr" { print $10; exit }')"; \
    if [ "$ACTUAL_FINGERPRINT" != "${EXPECTED_FINGERPRINT}" ]; then \
        echo "ERROR: ClamAV signing key fingerprint mismatch!" >&2; \
        exit 1; \
    fi; \
    gpg --batch --import "${CLAMAV_GPG_FILE}"; \
    gpg --batch --verify "${CLAMAV_SIG_FILE}" "${CLAMAV_DEB_FILE}"; \
    mv "${CLAMAV_DEB_FILE}" clamav.deb

FROM moby/buildkit:rootless AS buildkit

FROM ghcr.io/actions/actions-runner:latest AS skopeo

ARG SKOPEO_GIT_URL=https://github.com/containers/skopeo

USER root
RUN apt update \
    && apt install -y --no-install-recommends \
       ca-certificates \
       wget \
       tar \
       gnupg \
       curl \
       git \
       build-essential \
       libgpgme-dev \
       libassuan-dev \
       libbtrfs-dev \
       pkg-config \
       make \
       golang-go \
    && rm -rf /var/lib/apt/lists/*

RUN git clone "${SKOPEO_GIT_URL}" /src/go.podman.io/skopeo; \
    cd /src/go.podman.io/skopeo \
    && make bin/skopeo; \
    install -m 755 bin/skopeo /usr/bin/skopeo

FROM ubuntu:latest AS trivy
ARG TRIVY_BIN_URL=https://github.com/aquasecurity/trivy/releases/download/v0.76.0/
ARG TRIVY_BIN_FILE=trivy_0.76.0_Linux-64bit.tar.gz
ARG TRIVY_CHECKSUM_URL=https://github.com/aquasecurity/trivy/releases/download/v0.76.0/
ARG TRIVY_CHECKSUM_FILE=trivy_0.76.0_checksums.txt

RUN apt update \
    && apt install -y --no-install-recommends \
       ca-certificates \
       wget \
       tar

RUN set -eu; \
    BIN="${TRIVY_BIN_URL}${TRIVY_BIN_FILE}"; \
    CHECKSUMS="${TRIVY_CHECKSUM_URL}${TRIVY_CHECKSUM_FILE}"; \
    wget -O "${TRIVY_BIN_FILE}" "$BIN"; \
    wget -O "${TRIVY_CHECKSUM_FILE}" "$CHECKSUMS"; \
    checksum_line=$(grep -F " ${TRIVY_BIN_FILE}" "${TRIVY_CHECKSUM_FILE}" || true); \
    if [ -z "$checksum_line" ]; then \
        echo "ERROR: Checksum for ${TRIVY_BIN_FILE} not found." >&2; \
        exit 1; \
    fi; \
    printf '%s\n' "$checksum_line" | sha256sum --check -; \
    tar -xvf "${TRIVY_BIN_FILE}"; \
    install -m 755 trivy /usr/bin/trivy;

FROM ghcr.io/actions/actions-runner:latest

USER root

COPY --from=buildkit /usr/bin/buildctl /usr/local/bin/buildctl
COPY --from=skopeo /usr/bin/skopeo /usr/local/bin/skopeo
COPY --from=trivy /usr/bin/trivy /usr/local/bin/trivy
COPY --from=clamav /tmp/clamav.deb clamav.deb

RUN apt update \
    && apt install -y --no-install-recommends \
       ./clamav.deb \
       ca-certificates \
       libgpgme-dev \
    && rm -rf /var/lib/apt/lists/* ./clamav.deb

RUN groupadd clamav; \
    usermod -aG clamav runner; \
    mkdir -p /usr/local/etc; \
    mkdir -p /usr/local/share/clamav; \
    chown -R runner:clamav /usr/local/share/clamav

COPY freshclam.conf /usr/local/etc/freshclam.conf
COPY policy.json /usr/share/containers/policy.json

USER runner
WORKDIR /home/runner
ENV BUILDKIT_HOST=tcp://127.0.0.1:1234
ENTRYPOINT [ "/bin/bash", "-c", "sleep infinity" ]