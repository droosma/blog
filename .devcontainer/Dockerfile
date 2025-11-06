ARG NODE_VERSION=16
FROM mcr.microsoft.com/devcontainers/javascript-node:${NODE_VERSION}

ARG VARIANT=hugo
ARG VERSION=0.127.0

# Download Hugo
RUN apt-get update && apt-get install -y ca-certificates openssl git curl && \
    rm -rf /var/lib/apt/lists/* && \
    echo ${VERSION} && \
    case $(uname -m) in \
    aarch64) \
    export ARCH=ARM64 ;; \
    *) \
    export ARCH=64bit ;; \
    esac && \
    echo ${ARCH} && \
    wget -O ${VERSION}.tar.gz https://github.com/gohugoio/hugo/releases/download/v${VERSION}/${VARIANT}_${VERSION}_Linux-${ARCH}.tar.gz && \
    tar xf ${VERSION}.tar.gz && \
    mv hugo /usr/bin/hugo

# Hugo dev server port
EXPOSE 1313
