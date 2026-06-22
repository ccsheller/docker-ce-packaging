ARG GO_VERSION=1.24.8

FROM ghcr.io/loong64/golang:${GO_VERSION}-trixie AS builder

ARG RUNC_VERSION=v1.3.0
ARG CONTAINERD_VERSION=v1.7.28
ARG DOCKER_VERSION=v28.5.1
ARG DOCKER_GITCOMMIT=f8215cc266744ef195a50a70d427c345da2acdbb
ARG TINI_VERSION=v0.19.0
ARG DEBUG=0

RUN set -ex; \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    apt-get update; \
    apt-get install -y wget g++ make pkg-config patch file binutils libseccomp-dev libbtrfs-dev libdevmapper-dev; \
    rm -rf /var/lib/apt/lists/*; \
    printf '%s\n' \
        '#!/bin/sh' \
        'case "$1" in' \
        '  diff|status) exit 0 ;;' \
        '  describe) echo "unknown"; exit 0 ;;' \
        '  rev-parse)' \
        '    case "$2" in' \
        '      --show-toplevel) pwd ;;' \
        '      --short) echo "unknown" ;;' \
        '      *) echo "0000000000000000000000000000000000000000" ;;' \
        '    esac' \
        '    exit 0 ;;' \
        '  log) echo "0000000000000000000000000000000000000000"; exit 0 ;;' \
        'esac' \
        'echo "unsupported git command: $*" >&2' \
        'exit 1' \
        > /usr/local/bin/git; \
    chmod +x /usr/local/bin/git

WORKDIR /go/src/github.com/opencontainers
RUN set -ex; \
    mkdir -p runc; \
    wget -O /tmp/runc.tar.gz https://github.com/opencontainers/runc/archive/refs/tags/${RUNC_VERSION}.tar.gz; \
    tar -tzf /tmp/runc.tar.gz >/dev/null; \
    tar -xzf /tmp/runc.tar.gz --strip-components=1 -C runc; \
    rm -f /tmp/runc.tar.gz

WORKDIR /go/src/github.com/opencontainers/runc
RUN set -ex; \
    if [ "${DEBUG}" = "1" ]; then export GOFLAGS="-gcflags=all=-N -l"; fi; \
    wget -qO- "https://github.com/loong64/containerd-packaging/raw/refs/heads/main/runc.patch" | patch -p1; \
    go get -u github.com/seccomp/libseccomp-golang@v0.10.1-0.20240814065753-28423ed7600d; \
    go mod vendor; \
    sed -i 's@--dirty @@g' Makefile; \
    make static; \
    ./runc -v

WORKDIR /go/src/github.com/containerd
RUN set -ex; \
    mkdir -p containerd; \
    wget -O /tmp/containerd.tar.gz https://github.com/containerd/containerd/archive/refs/tags/${CONTAINERD_VERSION}.tar.gz; \
    tar -tzf /tmp/containerd.tar.gz >/dev/null; \
    tar -xzf /tmp/containerd.tar.gz --strip-components=1 -C containerd; \
    rm -f /tmp/containerd.tar.gz

WORKDIR /go/src/github.com/containerd/containerd
RUN set -ex; \
    if [ "${DEBUG}" = "1" ]; then export GOFLAGS="-gcflags=all=-N -l"; fi; \
    sed -i 's@arm64 || mipsle@arm64 || loong64 || mipsle@g' vendor/github.com/cilium/ebpf/internal/endian_le.go; \
    sed -i 's@ppc64le riscv64@ppc64le riscv64 loong64@g' vendor/github.com/cilium/ebpf/internal/endian_le.go; \
    sed -i "s@--dirty='.m' @@g" Makefile; \
    sed -i 's@$(shell if ! git diff --no-ext-diff --quiet --exit-code; then echo .m; fi)@@g' Makefile; \
    make STATIC=True; \
    ./bin/containerd -v; \
    ./bin/ctr -v

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    mkdir -p cli; \
    wget -O /tmp/cli.tar.gz https://github.com/docker/cli/archive/refs/tags/${DOCKER_VERSION}.tar.gz; \
    tar -tzf /tmp/cli.tar.gz >/dev/null; \
    tar -xzf /tmp/cli.tar.gz --strip-components=1 -C cli; \
    rm -f /tmp/cli.tar.gz

WORKDIR /go/src/github.com/docker/cli
RUN set -ex; \
    if [ "${DEBUG}" = "1" ]; then export GOFLAGS="-gcflags=all=-N -l"; fi; \
    make; \
    ./build/docker -v

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    mkdir -p docker; \
    wget -O /tmp/moby.tar.gz https://github.com/moby/moby/archive/refs/tags/${DOCKER_VERSION}.tar.gz; \
    tar -tzf /tmp/moby.tar.gz >/dev/null; \
    tar -xzf /tmp/moby.tar.gz --strip-components=1 -C docker; \
    rm -f /tmp/moby.tar.gz

WORKDIR /go/src/github.com/docker/docker
RUN set -ex; \
    if [ "${DEBUG}" = "1" ]; then export GOFLAGS="-gcflags=all=-N -l" DOCKER_GCFLAGS="all=-N -l"; fi; \
    mkdir bin; \
    VERSION=${DOCKER_VERSION#*v} DOCKER_GITCOMMIT=${DOCKER_GITCOMMIT} ./hack/make.sh binary-daemon; \
    ./bundles/binary-daemon/dockerd -v; \
    cp -rf bundles/binary-daemon bin/; \
    VERSION=${DOCKER_VERSION#*v} DOCKER_GITCOMMIT=${DOCKER_GITCOMMIT} ./hack/make.sh binary-proxy; \
    ./bundles/binary-proxy/docker-proxy --version; \
    cp -rf bundles/binary-proxy bin/

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    mkdir -p tini; \
    wget -O /tmp/tini.tar.gz https://github.com/krallin/tini/archive/refs/tags/${TINI_VERSION}.tar.gz; \
    tar -tzf /tmp/tini.tar.gz >/dev/null; \
    tar -xzf /tmp/tini.tar.gz --strip-components=1 -C tini; \
    rm -f /tmp/tini.tar.gz

WORKDIR /go/src/github.com/docker/tini
RUN set -ex; \
    tini_version="${TINI_VERSION#v}"; \
    printf '#define TINI_VERSION "%s"\n#define TINI_GIT ""\n#define TINI_MINIMAL 0\n' "${tini_version}" > src/tiniConfig.h; \
    printf 'static const char LICENSE[] = "";\nstatic const unsigned int LICENSE_len = 0;\n' > src/tiniLicense.h; \
    if [ "${DEBUG}" = "1" ]; then \
        gcc -static -O0 -g -DPR_SET_CHILD_SUBREAPER=36 -DPR_GET_CHILD_SUBREAPER=37 -o tini-static src/tini.c; \
    else \
        gcc -static -Os -s -DPR_SET_CHILD_SUBREAPER=36 -DPR_GET_CHILD_SUBREAPER=37 -o tini-static src/tini.c; \
    fi; \
    ./tini-static --version

WORKDIR /opt/docker
RUN set -ex; \
    cp /go/src/github.com/opencontainers/runc/runc /opt/docker/; \
    cp /go/src/github.com/containerd/containerd/bin/containerd /opt/docker/; \
    cp /go/src/github.com/containerd/containerd/bin/containerd-shim-runc-v2 /opt/docker/; \
    cp /go/src/github.com/containerd/containerd/bin/ctr /opt/docker/; \
    cp /go/src/github.com/docker/cli/build/docker /opt/docker/; \
    cp /go/src/github.com/docker/docker/bin/binary-daemon/dockerd /opt/docker/; \
    cp /go/src/github.com/docker/docker/bin/binary-proxy/docker-proxy /opt/docker/; \
    cp /go/src/github.com/docker/tini/tini-static /opt/docker/docker-init

WORKDIR /opt
RUN set -ex; \
    suffix=""; \
    if [ "${DEBUG}" = "1" ]; then suffix="-debug"; fi; \
    chmod +x docker/*; \
    for f in docker/*; do \
        echo "== ${f} =="; \
        file "${f}"; \
        if readelf -l "${f}" | grep -q 'Requesting program interpreter'; then \
            echo "${f} has a dynamic loader"; \
            exit 1; \
        fi; \
        if ! ldd "${f}" 2>&1 | grep -Eq 'not a dynamic executable|不是动态可执行文件'; then \
            ldd "${f}" || true; \
            echo "${f} is not fully static"; \
            exit 1; \
        fi; \
    done; \
    tar -czf docker-${DOCKER_VERSION#*v}${suffix}.tgz docker; \
    echo $(md5sum docker-${DOCKER_VERSION#*v}${suffix}.tgz) > docker-${DOCKER_VERSION#*v}${suffix}.tgz.md5; \
    rm -rf docker

FROM scratch AS artifact

COPY --from=builder /opt/docker-*.tgz /
COPY --from=builder /opt/docker-*.tgz.md5 /
