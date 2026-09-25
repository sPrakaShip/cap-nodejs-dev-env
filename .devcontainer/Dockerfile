# syntax=docker/dockerfile:1

ARG VARIANT="24"
FROM mcr.microsoft.com/devcontainers/javascript-node:${VARIANT}

# Install some generally useful tools
RUN apt-get update \
  && apt-get -y install --no-install-recommends \
  curl \
  entr \
  git \
  sqlite3 \
  watch \
  yq \
  && rm -rf /var/lib/apt/lists/*

# Install cf CLI v8 from the release tarball; the Cloud Foundry apt repo's
# signing key has an expired primary key, which Debian trixie's apt rejects
ARG TARGETARCH
RUN case "${TARGETARCH:-amd64}" in \
    amd64) CF_RELEASE=linux64-binary ;; \
    arm64) CF_RELEASE=linuxarm64-binary ;; \
    *) echo "Unsupported architecture: ${TARGETARCH}" && exit 1 ;; \
  esac \
  && curl -fsSL "https://packages.cloudfoundry.org/stable?release=${CF_RELEASE}&version=v8&source=github" \
  | tar -xz -C /usr/local/bin cf8 cf \
  && cf version

# Install latest SAP CAP SDK globally
USER node
RUN npm install -g \
  @sap/cds-dk@latest

WORKDIR /home/node
