# syntax=docker/dockerfile:1
# ─────────────────────────────────────────────────────────────────────────────
# Kubernetes node image
#
# Builds on top of a plain Ubuntu base with the chosen K8s distribution,
# multiple container runtimes (runc, gvisor/runsc pre-installed; kata-containers
# and wasmedge installed at first boot by k8s-runtime-install.service),
# Longhorn/Rook-Ceph storage prerequisites, and etcd backup/restore scripts.
#
# This image is exported and overlaid onto a bootable Ubuntu cloud image
# (which provides the kernel, bootloader, and kernel modules).
#
# Build args:
#   DISTRO     – kubeadm (default), k3s, k0s
#   DISTRO_VERSION – version for k3s or k0s, e.g. v1.30 (defaults to same as K8S_VERSION)
#   UBUNTU_VERSION – Ubuntu base version, e.g. 24.04 (default)
#   KATA_VERSION   – kata-containers release to bake into the first-boot installer
#   WASM_VERSION   – containerd/runwasi wasmedge release to bake into the first-boot installer
# ─────────────────────────────────────────────────────────────────────────────

ARG UBUNTU_VERSION=24.04
ARG K8S_VERSION
ARG DISTRO=kubeadm
ARG DISTRO_VERSION

FROM ubuntu:${UBUNTU_VERSION} AS base

ARG K8S_VERSION
ARG DISTRO

ENV DEBIAN_FRONTEND=noninteractive \
  K8S_VERSION=${K8S_VERSION}

# Validate required build args
RUN [ -n "${K8S_VERSION}" ] || { \
      echo "ERROR: K8S_VERSION build arg is required, e.g. --build-arg K8S_VERSION=v1.30" >&2; \
      exit 1; \
    }
RUN case "${DISTRO}" in \
      kubeadm|k3s|k0s) ;; \
  *) echo "ERROR: DISTRO must be one of: kubeadm, k3s, k0s (got: ${DISTRO})" >&2; exit 1 ;; \
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
    apt-get autoremove --purge -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /var/cache/debconf/*-old /var/log/apt /var/log/*.log

# ── Locale / doc / cache / unused data cleanup ────────────────────────────────
# Remove localisation data, man pages, documentation, and other data not needed
# on a headless Kubernetes node to reduce image size.
RUN rm -rf \
      /usr/share/locale/* \
      /usr/share/man/* \
      /usr/share/doc/* \
      /usr/share/info/* \
      /usr/share/zoneinfo/posix \
      /usr/share/zoneinfo/right \
      /usr/share/i18n \
      /usr/share/X11 \
      /var/cache/debconf/*-old \
      /var/cache/apt/archives/*.deb \
      /var/cache/ldconfig/* \
      /var/log/*.log \
      /var/log/apt

# ── gvisor (runsc sandbox runtime) ────────────────────────────────────────────
RUN curl -fsSL https://gvisor.dev/archive.key | \
      gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg && \
    printf 'deb [arch=%s signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main\n' \
      "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/gvisor.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends runsc && \
    apt-get autoremove --purge -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /var/cache/debconf/*-old /var/log/apt /var/log/*.log

# Shared config templates used by the distro-specific base stages below.
COPY config/containerd-config.toml /tmp/containerd-config.toml
COPY config/k3s-containerd.toml.tmpl /tmp/k3s-containerd.toml.tmpl

# ── kubeadm base ───────────────────────────────────────────────────────────────
FROM base AS kubeadm-base

ARG K8S_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  DISTRO=kubeadm \
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
  apt-get autoremove --purge -y && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* /var/cache/debconf/*-old /var/log/apt /var/log/*.log && \
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
  apt-get autoremove --purge -y; \
  apt-get clean; \
  rm -rf /var/lib/apt/lists/* /var/cache/debconf/*-old /var/log/apt /var/log/*.log; \
  systemctl enable --root=/ kubelet 2>/dev/null || true

# ── k3s base ───────────────────────────────────────────────────────────────────
FROM base AS k3s-base

ARG K8S_VERSION
ARG DISTRO_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  DISTRO=k3s \
  ETCD_CA_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
  ETCD_CERT_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt \
  ETCD_KEY_FILE=/var/lib/rancher/k3s/server/tls/etcd/server-client.key \
  ETCD_DATA_DIR=/var/lib/rancher/k3s/server/db/etcd \
  ETCD_BACKUP_DIR=/var/lib/rancher/k3s/server/db/etcd/backup \
  K8S_SERVICE=k3s

ARG K3S_VERSION=${DISTRO_VERSION}
RUN set -eu; \
  curl -fsSL https://get.k3s.io | INSTALL_K3S_VERSION="${K3S_VERSION}" INSTALL_K3S_SKIP_ENABLE=true INSTALL_K3S_SKIP_START=true sh -; \
  mkdir -p /var/lib/rancher/k3s/agent/etc/containerd; \
  cp /tmp/k3s-containerd.toml.tmpl /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl

# ── k0s base ───────────────────────────────────────────────────────────────────
FROM base AS k0s-base

ARG K8S_VERSION
ARG DISTRO_VERSION
ENV K8S_VERSION=${K8S_VERSION} \
  DISTRO=k0s \
  ETCD_CA_FILE=/var/lib/k0s/pki/etcd/ca.crt \
  ETCD_CERT_FILE=/var/lib/k0s/pki/etcd/server.crt \
  ETCD_KEY_FILE=/var/lib/k0s/pki/etcd/server.key \
  ETCD_DATA_DIR=/var/lib/k0s/etcd \
  ETCD_BACKUP_DIR=/var/lib/k0s/etcd/backup \
  K8S_SERVICE=k0scontroller

ARG K0S_VERSION=${DISTRO_VERSION}
RUN curl -fsSL https://get.k0s.sh | K0S_VERSION="${K0S_VERSION}" sh

# ── Final image selected by distro ─────────────────────────────────────────────
FROM ${DISTRO}-base AS final

ARG K8S_VERSION
ARG DISTRO
ARG KATA_VERSION
ARG WASM_VERSION

ENV DEBIAN_FRONTEND=noninteractive \
  K8S_VERSION=${K8S_VERSION} \
  DISTRO=${DISTRO}

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
  < /tmp/etcd-backup.sh.tmpl > /usr/local/bin/etcd-backup.sh; \
  envsubst '$ETCD_CA_FILE $ETCD_CERT_FILE $ETCD_KEY_FILE $ETCD_DATA_DIR $ETCD_BACKUP_DIR $K8S_SERVICE' \
      < /tmp/etcd-restore.sh.tmpl > /usr/local/bin/etcd-restore.sh; \
    chmod +x /usr/local/bin/etcd-backup.sh /usr/local/bin/etcd-restore.sh; \
    rm /tmp/etcd-backup.sh.tmpl /tmp/etcd-restore.sh.tmpl

# ── First-boot runtime installer ───────────────────────────────────────────────
# kata-containers and wasmedge are too large to embed in the image.
# This systemd service downloads and installs them at first boot, before
# cloud-config runs, so runtimes are ready when cloud-init configures the node.
COPY scripts/install-runtimes.sh.tmpl /tmp/install-runtimes.sh.tmpl
COPY config/k8s-runtime-install.service /etc/systemd/system/k8s-runtime-install.service
RUN set -eu; \
  KATA_VERSION="${KATA_VERSION}" WASM_VERSION="${WASM_VERSION}" \
  envsubst '$KATA_VERSION $WASM_VERSION' \
    < /tmp/install-runtimes.sh.tmpl > /usr/local/bin/k8s-runtime-install.sh; \
  chmod +x /usr/local/bin/k8s-runtime-install.sh; \
  rm /tmp/install-runtimes.sh.tmpl; \
  systemctl enable --root=/ k8s-runtime-install.service 2>/dev/null || true

# ── Final cleanup ──────────────────────────────────────────────────────────────
# Purge build-only tools no longer needed after all repos and packages are set up,
# configure journald for volatile storage, and remove any remaining caches/logs.
# gnupg/gpg/apt-transport-https are only needed to add APT repositories; they are
# not present in k3s/k0s stages (which don't add repos), so we tolerate absence.
RUN if dpkg -l gnupg gpg apt-transport-https 2>/dev/null | grep -q '^ii'; then \
      apt-get purge -y --auto-remove gnupg gpg apt-transport-https; \
    else \
      echo "INFO: build-only tools not installed in this stage, skipping purge"; \
    fi; \
    apt-get clean; \
    rm -rf \
      /var/lib/apt/lists/* \
      /var/cache/debconf/*-old \
      /var/cache/apt/archives/*.deb \
      /var/cache/ldconfig/* \
      /var/log/apt \
      /var/log/*.log \
      /var/log/journal/* \
      /tmp/* \
      /var/tmp/*

RUN mkdir -p /etc/systemd && \
    printf '[Journal]\nStorage=volatile\n' > /etc/systemd/journald.conf
