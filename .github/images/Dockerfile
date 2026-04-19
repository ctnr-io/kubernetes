# syntax=docker/dockerfile:1
# ─────────────────────────────────────────────────────────────────────────────
# Kubernetes node image
#
# Builds on top of a plain Ubuntu base with the chosen K8s distribution,
# multiple container runtimes (runc, gvisor/runsc, kata-containers, wasmtime,
# wasmedge), Longhorn/Rook-Ceph storage prerequisites, and etcd backup/restore
# scripts pre-installed.
#
# This image is exported and overlaid onto a bootable Ubuntu cloud image
# (which provides the kernel, bootloader, and kernel modules).
#
# Build args:
#   K8S_VERSION    – required, e.g. v1.30 or v1.33
#   K8S_DISTRO     – kubeadm (default), k3s, k0s
#   UBUNTU_VERSION – Ubuntu base version, e.g. 24.04 (default)
# ─────────────────────────────────────────────────────────────────────────────

ARG UBUNTU_VERSION=24.04
ARG K8S_VERSION
ARG K8S_DISTRO=kubeadm

FROM ubuntu:${UBUNTU_VERSION} AS base

ARG K8S_VERSION
ARG K8S_DISTRO

ENV DEBIAN_FRONTEND=noninteractive \
  K8S_VERSION=${K8S_VERSION}

# Validate required build args
RUN [ -n "${K8S_VERSION}" ] || { \
      echo "ERROR: K8S_VERSION build arg is required, e.g. --build-arg K8S_VERSION=v1.30" >&2; \
      exit 1; \
    }
RUN case "${K8S_DISTRO}" in \
      kubeadm|k3s|k0s) ;; \
      *) echo "ERROR: K8S_DISTRO must be one of: kubeadm, k3s, k0s (got: ${K8S_DISTRO})" >&2; exit 1 ;; \
    esac

# ── Static config files ────────────────────────────────────────────────────────
COPY config/modules-load.conf   /etc/modules-load.d/k8s.conf
COPY config/sysctl-k8s.conf     /etc/sysctl.d/k8s.conf
COPY config/sysctl-inotify.conf /etc/sysctl.d/99-max-open-files.conf
COPY config/sysctl-vm.conf      /etc/sysctl.d/99-max-memory-mapped-files.conf

# ── Base packages ──────────────────────────────────────────────────────────────
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      sudo \
      ca-certificates \
      curl \
      gnupg \
      apt-transport-https \
      gpg \
      gettext-base \
      grepcidr \
      etcd-client \
      nfs-common \
      open-iscsi \
      util-linux \
      lvm2 \
      cryptsetup \
      zstd \
      jq && \
    rm -rf /var/lib/apt/lists/*

# ── Locale / doc / cache cleanup ──────────────────────────────────────────────
# Remove localisation data, man pages, and documentation to reduce image size.
# None of these are needed on a headless Kubernetes node.
RUN rm -rf \
      /usr/share/locale/* \
      /usr/share/man/* \
      /usr/share/doc/* \
      /usr/share/info/* \
      /var/cache/debconf/*-old \
      /var/cache/apt/archives/*.deb \
      /var/log/*.log \
      /var/log/apt

# ── gvisor (runsc sandbox runtime) ────────────────────────────────────────────
RUN curl -fsSL https://gvisor.dev/archive.key | \
      gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg && \
    printf 'deb [arch=%s signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main\n' \
      "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/gvisor.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends runsc && \
    rm -rf /var/lib/apt/lists/*

# ── Kata Containers (hardware-isolated sandbox, requires nested virt) ──────────
# Use the GitHub web redirect to discover the latest version instead of parsing
# the releases API JSON, which often contains control characters in the release
# body that cause jq to exit with a parse error (exit code 5).
RUN set -eu; \
    KATA_VERSION=$(curl -fsSL -o /dev/null -w '%{url_effective}' \
      "https://github.com/kata-containers/kata-containers/releases/latest" \
      | sed 's|.*/||'); \
    [ -n "${KATA_VERSION}" ] || { \
      echo "ERROR: could not determine kata-containers latest version" >&2; \
      exit 1; \
    }; \
    echo "Installing Kata Containers ${KATA_VERSION}..."; \
    BASE_URL="https://github.com/kata-containers/kata-containers/releases/download/${KATA_VERSION}"; \
    ZST_NAME="kata-static-${KATA_VERSION}-amd64.tar.zst"; \
    XZ_NAME="kata-static-${KATA_VERSION}-amd64.tar.xz"; \
    if curl --head -fsSL "${BASE_URL}/${ZST_NAME}" > /dev/null 2>&1; then \
      curl -fsSL "${BASE_URL}/${ZST_NAME}" | tar --zstd -xf - -C /; \
    elif curl --head -fsSL "${BASE_URL}/${XZ_NAME}" > /dev/null 2>&1; then \
      curl -fsSL "${BASE_URL}/${XZ_NAME}" | tar -xJf - -C /; \
    else \
      echo "ERROR: kata-containers amd64 static tarball not found for ${KATA_VERSION}" >&2; \
      exit 1; \
    fi; \
    ln -sf /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/containerd-shim-kata-v2

# ── WASM runtimes (wasmtime and wasmedge via containerd/runwasi) ───────────────
RUN --mount=type=secret,id=github_token \
    set -eu; \
    GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
    if [ -n "${GH_TOKEN}" ]; then \
      RUNWASI_RELEASE=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
        "https://api.github.com/repos/containerd/runwasi/releases/latest" \
        | tr -d '\000-\037'); \
    else \
      RUNWASI_RELEASE=$(curl -fsSL \
        "https://api.github.com/repos/containerd/runwasi/releases/latest" \
        | tr -d '\000-\037'); \
    fi; \
    for SHIM in wasmtime wasmedge; do \
      URL=$(echo "$RUNWASI_RELEASE" | jq -r --arg shim "$SHIM" \
        '.assets[] | select(.name | test("containerd-shim-" + $shim + ".*linux.*amd64\\.tar\\.gz$")) | .browser_download_url' \
        | head -1); \
      if [ -n "${URL}" ]; then \
        echo "Installing containerd-shim-${SHIM}..."; \
        curl -fsSL "${URL}" | tar xz -C /usr/local/bin; \
      else \
        echo "WARNING: containerd-shim-${SHIM} not found in runwasi releases, skipping."; \
      fi; \
    done

# Shared config templates used by the distro-specific base stages below.
COPY config/containerd-config.toml /tmp/containerd-config.toml
COPY config/k3s-containerd.toml.tmpl /tmp/k3s-containerd.toml.tmpl

# ── kubeadm base ───────────────────────────────────────────────────────────────
FROM base AS kubeadm-base

ARG K8S_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  K8S_DISTRO=kubeadm \
  ETCD_CA_FILE=/etc/kubernetes/pki/etcd/ca.crt \
  ETCD_CERT_FILE=/etc/kubernetes/pki/etcd/server.crt \
  ETCD_KEY_FILE=/etc/kubernetes/pki/etcd/server.key \
  ETCD_DATA_DIR=/var/lib/etcd \
  ETCD_BACKUP_DIR=/var/lib/etcd/backup \
  K8S_SERVICE=kubelet

RUN install -m 0755 -d /etc/apt/keyrings && \
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor --batch --no-tty -o /etc/apt/keyrings/docker.gpg && \
  chmod a+r /etc/apt/keyrings/docker.gpg && \
  printf 'deb [arch=%s signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu %s stable\n' \
  "$(dpkg --print-architecture)" "$(. /etc/os-release && echo "$VERSION_CODENAME")" \
  > /etc/apt/sources.list.d/docker.list && \
  apt-get update && \
  apt-get install -y --no-install-recommends containerd.io && \
  rm -rf /var/lib/apt/lists/* && \
  mkdir -p /etc/containerd && \
  cp /tmp/containerd-config.toml /etc/containerd/config.toml && \
  systemctl enable --root=/ containerd 2>/dev/null || true

RUN set -eu; \
  install -m 0755 -d /etc/apt/keyrings; \
  curl -fsSL "https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key" | \
  gpg --dearmor --batch --no-tty -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg; \
  printf 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/%s/deb/ /\n' \
  "${K8S_VERSION}" > /etc/apt/sources.list.d/kubernetes.list; \
  apt-get update; \
  apt-get install -y --no-install-recommends kubelet kubeadm kubectl; \
  apt-mark hold kubelet kubeadm kubectl; \
  rm -rf /var/lib/apt/lists/*; \
  systemctl enable --root=/ kubelet 2>/dev/null || true

# ── k3s base ───────────────────────────────────────────────────────────────────
FROM base AS k3s-base

ARG K8S_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  K8S_DISTRO=k3s \
  ETCD_CA_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  ETCD_CERT_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt \
  ETCD_KEY_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-client.key \
  ETCD_DATA_DIR=/var/lib/rancher/k3s/server/db/etcd \
  ETCD_BACKUP_DIR=/var/lib/rancher/k3s/server/db/etcd/backup \
  K8S_SERVICE=k3s

RUN --mount=type=secret,id=github_token \
  set -eu; \
  GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
  if [ -n "${GH_TOKEN}" ]; then \
  K3S_RELEASES=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
  "https://api.github.com/repos/k3s-io/k3s/releases" \
  | tr -d '\000-\037'); \
  else \
  K3S_RELEASES=$(curl -fsSL "https://api.github.com/repos/k3s-io/k3s/releases" \
  | tr -d '\000-\037'); \
  fi; \
  K3S_VERSION=$(echo "${K3S_RELEASES}" | \
  jq -r --arg prefix "${K8S_VERSION}." \
  '[.[] | select(.tag_name | startswith($prefix)) | select(.prerelease == false)] | .[0].tag_name // empty'); \
  if [ -z "${K3S_VERSION}" ]; then \
  echo "WARNING: No k3s stable release found for ${K8S_VERSION}. k3s will not be installed." >&2; \
  else \
  echo "Installing k3s ${K3S_VERSION}..."; \
  curl -fsSL https://get.k3s.io | INSTALL_K3S_VERSION="${K3S_VERSION}" INSTALL_K3S_SKIP_START=true sh -; \
  mkdir -p /var/lib/rancher/k3s/agent/etc/containerd; \
  cp /tmp/k3s-containerd.toml.tmpl /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl; \
  fi

# ── k0s base ───────────────────────────────────────────────────────────────────
FROM base AS k0s-base

ARG K8S_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  K8S_DISTRO=k0s \
  ETCD_CA_FILE=/var/lib/k0s/pki/etcd/ca.crt \
  ETCD_CERT_FILE=/var/lib/k0s/pki/etcd/server.crt \
  ETCD_KEY_FILE=/var/lib/k0s/pki/etcd/server.key \
  ETCD_DATA_DIR=/var/lib/k0s/etcd \
  ETCD_BACKUP_DIR=/var/lib/k0s/etcd/backup \
  K8S_SERVICE=k0scontroller

RUN --mount=type=secret,id=github_token \
  set -eu; \
  GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
  if [ -n "${GH_TOKEN}" ]; then \
  K0S_RELEASES=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
  "https://api.github.com/repos/k0sproject/k0s/releases" \
  | tr -d '\000-\037'); \
  else \
  K0S_RELEASES=$(curl -fsSL "https://api.github.com/repos/k0sproject/k0s/releases" \
  | tr -d '\000-\037'); \
  fi; \
  K0S_VERSION=$(echo "${K0S_RELEASES}" | \
  jq -r --arg prefix "${K8S_VERSION}." \
  '[.[] | select(.tag_name | startswith($prefix)) | select(.prerelease == false)] | .[0].tag_name // empty'); \
  if [ -z "${K0S_VERSION}" ]; then \
  echo "WARNING: No k0s stable release found for ${K8S_VERSION}. k0s will not be installed." >&2; \
  else \
  echo "Installing k0s ${K0S_VERSION}..."; \
  curl -fsSL https://get.k0s.sh | K0S_VERSION="${K0S_VERSION}" sh; \
  fi

# ── Final image selected by distro ─────────────────────────────────────────────
FROM ${K8S_DISTRO}-base AS final

ARG K8S_VERSION
ARG K8S_DISTRO

ENV DEBIAN_FRONTEND=noninteractive \
  K8S_VERSION=${K8S_VERSION} \
  K8S_DISTRO=${K8S_DISTRO}

# ── Storage services ───────────────────────────────────────────────────────────
# lvm2 and cryptsetup (Longhorn/Rook-Ceph) were installed with base packages.
# Enable iscsid at boot for iSCSI-backed volumes.
RUN systemctl enable --root=/ iscsid 2>/dev/null || true

# ── etcd backup/restore scripts ────────────────────────────────────────────────
# Pre-installed on all nodes; the daily cron is activated by cloud-init only on
# control-plane nodes. The selected distro base stage provides the etcd paths,
# data directory, backup directory, and service name used here.
COPY scripts/etcd-backup.sh.tmpl  /tmp/etcd-backup.sh.tmpl
COPY scripts/etcd-restore.sh.tmpl /tmp/etcd-restore.sh.tmpl
RUN set -eu; \
  envsubst '$ETCD_CA_FILE $ETCD_CERT_FILE $ETCD_KEY_FILE $ETCD_DATA_DIR $ETCD_BACKUP_DIR $K8S_SERVICE' \
      < /tmp/etcd-backup.sh.tmpl  > /usr/local/bin/etcd-backup.sh; \
  envsubst '$ETCD_CA_FILE $ETCD_CERT_FILE $ETCD_KEY_FILE $ETCD_DATA_DIR $ETCD_BACKUP_DIR $K8S_SERVICE' \
      < /tmp/etcd-restore.sh.tmpl > /usr/local/bin/etcd-restore.sh; \
    chmod +x /usr/local/bin/etcd-backup.sh /usr/local/bin/etcd-restore.sh; \
    rm /tmp/etcd-backup.sh.tmpl /tmp/etcd-restore.sh.tmpl
