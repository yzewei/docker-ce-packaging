ARG GO_VERSION=1.24.3

FROM ghcr.io/loong64/golang:${GO_VERSION}-trixie AS builder

ARG DOCKER_VERSION
ARG TINI_VERSION=v0.19.0

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    git clone -b ${DOCKER_VERSION} https://github.com/moby/moby docker --depth=1

# 提取 RUNC_VERSION 和 CONTAINERD_VERSION
RUN set -ex; \
    export RUNC_VERSION=$(grep -oP 'RUNC_VERSION:=[v0-9.]+' docker/hack/dockerfile/install/runc.installer | head -n1 | cut -d= -f2); \
    export CONTAINERD_VERSION=$(grep -oP 'CONTAINERD_VERSION:=[v0-9.]+' docker/hack/dockerfile/install/containerd.installer | head -n1 | cut -d= -f2); \
    echo "RUNC_VERSION=$RUNC_VERSION" > /versions.env; \
    echo "CONTAINERD_VERSION=$CONTAINERD_VERSION" >> /versions.env

RUN set -ex; \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    apt-get update; \
    apt-get install -y wget g++ cmake make pkg-config git libseccomp-dev libbtrfs-dev libdevmapper-dev; \
    rm -rf /var/lib/apt/lists/*

WORKDIR /go/src/github.com/opencontainers
RUN set -ex; \
    . /versions.env; \
    git clone -b ${RUNC_VERSION} https://github.com/opencontainers/runc --depth=1

WORKDIR /go/src/github.com/opencontainers/runc
RUN set -ex; \
    curl -sSL "https://github.com/loong64/containerd-packaging/raw/refs/heads/main/runc.patch" | git apply; \
    go get -u github.com/seccomp/libseccomp-golang@v0.10.1-0.20240814065753-28423ed7600d; \
    go mod vendor; \
    sed -i 's@--dirty @@g' Makefile; \
    make static; \
    ./runc -v

WORKDIR /go/src/github.com/containerd
RUN set -ex; \
    . /versions.env; \
    git clone -b ${CONTAINERD_VERSION} https://github.com/containerd/containerd --depth=1

WORKDIR /go/src/github.com/containerd/containerd
RUN sed -i 's@arm64 || mipsle@arm64 || loong64 || mipsle@g' vendor/github.com/cilium/ebpf/internal/endian_le.go; \
    sed -i 's@ppc64le riscv64@ppc64le riscv64 loong64@g' vendor/github.com/cilium/ebpf/internal/endian_le.go; \
    sed -i "s@--dirty='.m' @@g" Makefile; \
    sed -i 's@$(shell if ! git diff --no-ext-diff --quiet --exit-code; then echo .m; fi)@@g' Makefile; \
    make STATIC=True; \
    ./bin/containerd -v; \
    ./bin/ctr -v

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    git clone -b ${DOCKER_VERSION} https://github.com/docker/cli --depth=1

WORKDIR /go/src/github.com/docker/cli
RUN make; \
    ./build/docker -v

WORKDIR /go/src/github.com/docker/docker
RUN mkdir bin; \
    VERSION=${DOCKER_VERSION#*v} ./hack/make.sh; \
    ./bundles/binary-daemon/dockerd -v; \
    cp -rf bundles/binary-daemon bin/; \
    VERSION=${DOCKER_VERSION#*v} ./hack/make.sh binary-proxy; \
    ./bundles/binary-proxy/docker-proxy --version; \
    cp -rf bundles/binary-proxy bin/

WORKDIR /go/src/github.com/docker
RUN set -ex; \
    git clone -b ${TINI_VERSION} https://github.com/krallin/tini --depth=1

WORKDIR /go/src/github.com/docker/tini
RUN cmake .; \
    make tini-static; \
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
    chmod +x docker/*; \
    tar -czf docker-${DOCKER_VERSION#*v}.tgz docker; \
    echo $(md5sum docker-${DOCKER_VERSION#*v}.tgz) > docker-${DOCKER_VERSION#*v}.tgz.md5; \
    rm -rf docker

FROM ghcr.io/loong64/debian:trixie-slim
ARG DOCKER_VERSION

COPY --from=builder /opt /opt
WORKDIR /opt

RUN set -ex; \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
    cat docker-${DOCKER_VERSION#*v}.tgz.md5

VOLUME /dist

CMD cp -f docker-* /dist/
