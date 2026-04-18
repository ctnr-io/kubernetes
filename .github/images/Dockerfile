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
FROM ubuntu:${UBUNTU_VERSION}

ARG K8S_VERSION
ARG K8S_DISTRO=kubeadm

ENV DEBIAN_FRONTEND=noninteractive \
    K8S_VERSION=${K8S_VERSION} \
    K8S_DISTRO=${K8S_DISTRO}

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

# ── gvisor (runsc sandbox runtime) ────────────────────────────────────────────
RUN curl -fsSL https://gvisor.dev/archive.key | \
      gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg && \
    printf 'deb [arch=%s signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main\n' \
      "$(dpkg --print-architecture)" > /etc/apt/sources.list.d/gvisor.list && \
    apt-get update && \
    apt-get install -y --no-install-recommends runsc && \
    rm -rf /var/lib/apt/lists/*

# ── Kata Containers (hardware-isolated sandbox, requires nested virt) ──────────
RUN --mount=type=secret,id=github_token \
    set -eu; \
    GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
    if [ -n "${GH_TOKEN}" ]; then \
      KATA_RELEASE=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
        "https://api.github.com/repos/kata-containers/kata-containers/releases/latest"); \
    else \
      KATA_RELEASE=$(curl -fsSL \
        "https://api.github.com/repos/kata-containers/kata-containers/releases/latest"); \
    fi; \
    KATA_VERSION=$(echo "$KATA_RELEASE" | jq -r '.tag_name'); \
    KATA_URL=$(echo "$KATA_RELEASE" | jq -r \
      '.assets[] | select(.name | test("kata-static-.*-amd64\\.tar\\.(xz|zst)$")) | .browser_download_url' \
      | head -1); \
    [ -n "${KATA_URL}" ] || { \
      echo "ERROR: kata-containers amd64 static tarball not found in release ${KATA_VERSION}" >&2; \
      exit 1; \
    }; \
    echo "Installing Kata Containers ${KATA_VERSION}..."; \
    if echo "${KATA_URL}" | grep -q '\.tar\.zst'; then \
      curl -fsSL "${KATA_URL}" | tar --zstd -xf - -C /; \
    else \
      curl -fsSL "${KATA_URL}" | tar -xJf - -C /; \
    fi; \
    ln -sf /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/containerd-shim-kata-v2

# ── WASM runtimes (wasmtime and wasmedge via containerd/runwasi) ───────────────
RUN --mount=type=secret,id=github_token \
    set -eu; \
    GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
    if [ -n "${GH_TOKEN}" ]; then \
      RUNWASI_RELEASE=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
        "https://api.github.com/repos/containerd/runwasi/releases/latest"); \
    else \
      RUNWASI_RELEASE=$(curl -fsSL \
        "https://api.github.com/repos/containerd/runwasi/releases/latest"); \
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

# ── containerd.io + config (kubeadm only) ─────────────────────────────────────
# k3s and k0s bundle their own containerd and pick up extra runtimes via their
# own config template mechanism below.
COPY config/containerd-config.toml /tmp/containerd-config.toml
RUN if [ "${K8S_DISTRO}" = "kubeadm" ]; then \
      install -m 0755 -d /etc/apt/keyrings && \
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
      systemctl enable --root=/ containerd 2>/dev/null || true; \
    fi; \
    rm -f /tmp/containerd-config.toml

# ── Kubernetes distribution ────────────────────────────────────────────────────
# k3s uses a containerd config template merged at startup to register extra runtimes.
COPY config/k3s-containerd.toml.tmpl /tmp/k3s-containerd.toml.tmpl
RUN --mount=type=secret,id=github_token \
    set -eu; \
    GH_TOKEN=$(cat /run/secrets/github_token 2>/dev/null || true); \
    case "${K8S_DISTRO}" in \
    \
    kubeadm) \
      install -m 0755 -d /etc/apt/keyrings; \
      curl -fsSL "https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key" | \
        gpg --dearmor --batch --no-tty -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg; \
      printf 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/%s/deb/ /\n' \
        "${K8S_VERSION}" > /etc/apt/sources.list.d/kubernetes.list; \
      apt-get update; \
      apt-get install -y --no-install-recommends kubelet kubeadm kubectl; \
      apt-mark hold kubelet kubeadm kubectl; \
      rm -rf /var/lib/apt/lists/*; \
      systemctl enable --root=/ kubelet 2>/dev/null || true; \
      ;; \
    \
    k3s) \
      if [ -n "${GH_TOKEN}" ]; then \
        K3S_RELEASES=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
          "https://api.github.com/repos/k3s-io/k3s/releases"); \
      else \
        K3S_RELEASES=$(curl -fsSL "https://api.github.com/repos/k3s-io/k3s/releases"); \
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
      fi; \
      ;; \
    \
    k0s) \
      if [ -n "${GH_TOKEN}" ]; then \
        K0S_RELEASES=$(curl -fsSL -H "Authorization: Bearer ${GH_TOKEN}" \
          "https://api.github.com/repos/k0sproject/k0s/releases"); \
      else \
        K0S_RELEASES=$(curl -fsSL "https://api.github.com/repos/k0sproject/k0s/releases"); \
      fi; \
      K0S_VERSION=$(echo "${K0S_RELEASES}" | \
        jq -r --arg prefix "${K8S_VERSION}." \
          '[.[] | select(.tag_name | startswith($prefix)) | select(.prerelease == false)] | .[0].tag_name // empty'); \
      if [ -z "${K0S_VERSION}" ]; then \
        echo "WARNING: No k0s stable release found for ${K8S_VERSION}. k0s will not be installed." >&2; \
      else \
        echo "Installing k0s ${K0S_VERSION}..."; \
        curl -fsSL https://get.k0s.sh | K0S_VERSION="${K0S_VERSION}" sh; \
      fi; \
      ;; \
    \
    esac; \
    rm -f /tmp/k3s-containerd.toml.tmpl

# ── Storage services ───────────────────────────────────────────────────────────
# lvm2 and cryptsetup (Longhorn/Rook-Ceph) were installed with base packages.
# Enable iscsid at boot for iSCSI-backed volumes.
RUN systemctl enable --root=/ iscsid 2>/dev/null || true

# ── etcd backup/restore scripts ────────────────────────────────────────────────
# Pre-installed on all nodes; the daily cron is activated by cloud-init only on
# control-plane nodes. Build-time variables (cert paths, data dir, service name)
# are expanded into the templates with envsubst (from the gettext-base package).
COPY scripts/etcd-backup.sh.tmpl  /tmp/etcd-backup.sh.tmpl
COPY scripts/etcd-restore.sh.tmpl /tmp/etcd-restore.sh.tmpl
RUN set -eu; \
    case "${K8S_DISTRO}" in \
    kubeadm) \
      export ETCD_CA_FILE="/etc/kubernetes/pki/etcd/ca.crt" \
             ETCD_CERT_FILE="/etc/kubernetes/pki/etcd/server.crt" \
             ETCD_KEY_FILE="/etc/kubernetes/pki/etcd/server.key" \
             ETCD_DATA_DIR="/var/lib/etcd" \
             K8S_SERVICE="kubelet"; \
      ;; \
    k3s) \
      export ETCD_CA_FILE="/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt" \
             ETCD_CERT_FILE="/var/lib/rancher/k3s/server/tls/etcd/server-client.crt" \
             ETCD_KEY_FILE="/var/lib/rancher/k3s/server/tls/etcd/server-client.key" \
             ETCD_DATA_DIR="/var/lib/rancher/k3s/server/db/etcd" \
             K8S_SERVICE="k3s"; \
      ;; \
    k0s) \
      export ETCD_CA_FILE="/var/lib/k0s/pki/etcd/ca.crt" \
             ETCD_CERT_FILE="/var/lib/k0s/pki/etcd/server.crt" \
             ETCD_KEY_FILE="/var/lib/k0s/pki/etcd/server.key" \
             ETCD_DATA_DIR="/var/lib/k0s/etcd" \
             K8S_SERVICE="k0scontroller"; \
      ;; \
    esac; \
    envsubst '$ETCD_CA_FILE $ETCD_CERT_FILE $ETCD_KEY_FILE $ETCD_DATA_DIR $K8S_SERVICE' \
      < /tmp/etcd-backup.sh.tmpl  > /usr/local/bin/etcd-backup.sh; \
    envsubst '$ETCD_CA_FILE $ETCD_CERT_FILE $ETCD_KEY_FILE $ETCD_DATA_DIR $K8S_SERVICE' \
      < /tmp/etcd-restore.sh.tmpl > /usr/local/bin/etcd-restore.sh; \
    chmod +x /usr/local/bin/etcd-backup.sh /usr/local/bin/etcd-restore.sh; \
    rm /tmp/etcd-backup.sh.tmpl /tmp/etcd-restore.sh.tmpl
