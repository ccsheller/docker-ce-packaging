ARG GO_VERSION=1.24.8
ARG COMPOSE_VERSION=v2.40.3

FROM ghcr.io/loong64/golang:${GO_VERSION}-trixie AS builder
ARG COMPOSE_VERSION

RUN set -eux; \
    apt-get update; \
    apt-get install -y wget make file binutils; \
    rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN set -eux; \
    mkdir -p compose; \
    wget -O /tmp/compose.tar.gz https://github.com/docker/compose/archive/refs/tags/${COMPOSE_VERSION}.tar.gz; \
    tar -tzf /tmp/compose.tar.gz >/dev/null; \
    tar -xzf /tmp/compose.tar.gz --strip-components=1 -C compose; \
    rm -f /tmp/compose.tar.gz

WORKDIR /src/compose
RUN set -eux; \
    version="${COMPOSE_VERSION#v}"; \
    GOOS=linux GOARCH=loong64 CGO_ENABLED=0 VERSION="${COMPOSE_VERSION}" make; \
    mkdir -p /out; \
    if [ -x bin/build/docker-compose ]; then \
        cp bin/build/docker-compose /out/docker-compose-linux-loong64-${version}; \
    elif [ -x bin/docker-compose ]; then \
        cp bin/docker-compose /out/docker-compose-linux-loong64-${version}; \
    else \
        find . -maxdepth 4 -type f -name 'docker-compose*' -ls; \
        echo 'docker-compose binary not found'; \
        exit 1; \
    fi; \
    chmod +x /out/docker-compose-linux-loong64-${version}; \
    /out/docker-compose-linux-loong64-${version} version; \
    file /out/docker-compose-linux-loong64-${version}; \
    if readelf -l /out/docker-compose-linux-loong64-${version} | grep -q 'Requesting program interpreter'; then \
        echo 'docker-compose has a dynamic loader'; \
        exit 1; \
    fi; \
    if ! ldd /out/docker-compose-linux-loong64-${version} 2>&1 | grep -Eq 'not a dynamic executable|不是动态可执行文件'; then \
        ldd /out/docker-compose-linux-loong64-${version} || true; \
        echo 'docker-compose is not fully static'; \
        exit 1; \
    fi; \
    md5sum /out/docker-compose-linux-loong64-${version} > /out/docker-compose-linux-loong64-${version}.md5

FROM scratch AS artifact
COPY --from=builder /out/ /
