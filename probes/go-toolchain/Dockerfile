ARG GO_VERSION=1.24.8

FROM ghcr.io/loong64/golang:${GO_VERSION}-trixie AS builder
ARG GO_VERSION=1.24.8

ARG DEBUG=0

RUN set -eux; \
    apt-get update; \
    apt-get install -y file binutils ca-certificates; \
    rm -rf /var/lib/apt/lists/*

WORKDIR /src

RUN printf '%s\n' \
    'package main' \
    '' \
    'import (' \
    '    "fmt"' \
    '    "net"' \
    '    "os"' \
    '    "runtime"' \
    ')' \
    '' \
    'func main() {' \
    '    fmt.Printf("hello from %s/%s with %s\\n", runtime.GOOS, runtime.GOARCH, runtime.Version())' \
    '    fmt.Printf("args: %v\\n", os.Args)' \
    '    ips, err := net.LookupHost("localhost")' \
    '    if err != nil {' \
    '        fmt.Printf("lookup localhost error: %v\\n", err)' \
    '        return' \
    '    }' \
    '    fmt.Printf("localhost: %v\\n", ips)' \
    '}' \
    > hello.go

RUN set -eux; \
    go version > /out-go-version.txt; \
    mkdir -p /out; \
    flags=""; \
    if [ "${DEBUG}" = "1" ]; then flags="-gcflags=all=-N -l"; fi; \
    CGO_ENABLED=0 GOOS=linux GOARCH=loong64 go build ${flags} -trimpath -ldflags='-s -w' -o /out/hello-purego ./hello.go; \
    CGO_ENABLED=0 GOOS=linux GOARCH=loong64 go build ${flags} -trimpath -tags 'netgo osusergo' -ldflags='-s -w' -o /out/hello-netgo ./hello.go; \
    /out/hello-purego; \
    /out/hello-netgo

WORKDIR /out

RUN set -eux; \
    { \
        echo '== go version =='; \
        cat /out-go-version.txt; \
        echo; \
        echo '== build args =='; \
        echo "GO_VERSION=${GO_VERSION}"; \
        echo "DEBUG=${DEBUG}"; \
        echo; \
        echo '== file =='; \
        file hello-purego hello-netgo; \
        echo; \
        echo '== readelf notes =='; \
        readelf -n hello-purego || true; \
    } > build-info.txt; \
    sha256sum hello-purego hello-netgo build-info.txt > sha256sums.txt; \
    tar -czf /go-toolchain-probe-loong64.tar.gz hello-purego hello-netgo build-info.txt sha256sums.txt

FROM scratch AS artifact

COPY --from=builder /go-toolchain-probe-loong64.tar.gz /
