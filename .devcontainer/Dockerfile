# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

# @author         Michael Roberts <michael@observerly.com>
# @package        tailgate
# @license        Copyright © 2026 Michael Roberts

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

# Go toolchain, gopls, dlv, golangci-lint v2 and staticcheck come with the base image.
FROM mcr.microsoft.com/devcontainers/go:2-1.27-trixie

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

ARG GOVULNCHECK_VERSION=v1.9.0
ARG GORELEASER_VERSION=v2.19.0

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

# govulncheck: built into /go/bin alongside the base image's tools. The throwaway GOPATH keeps
# the module cache and checksum database out of /go/pkg, which must not contain root-owned files.
RUN GOPATH=/tmp/gopath GOBIN=/go/bin GOCACHE=/tmp/gocache \
        go install golang.org/x/vuln/cmd/govulncheck@${GOVULNCHECK_VERSION} \
    && rm -rf /tmp/gopath /tmp/gocache

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

# GoReleaser: release tarball for the container's architecture, verified against checksums.txt.
RUN set -eu; \
    case "$(dpkg --print-architecture)" in \
        amd64) arch=x86_64 ;; \
        arm64) arch=arm64 ;; \
        *) echo "unsupported architecture: $(dpkg --print-architecture)" >&2; exit 1 ;; \
    esac; \
    url="https://github.com/goreleaser/goreleaser/releases/download/${GORELEASER_VERSION}"; \
    tarball="goreleaser_Linux_${arch}.tar.gz"; \
    cd /tmp; \
    curl -fsSLO "${url}/${tarball}"; \
    curl -fsSL "${url}/checksums.txt" | grep "  ${tarball}\$" | sha256sum -c -; \
    tar -xzf "${tarball}" -C /usr/local/bin goreleaser; \
    rm "${tarball}"

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #

# Mount points for the cache volumes in devcontainer.json. A new named volume copies the
# ownership of the directory it is mounted over, so these must exist and belong to vscode.
RUN mkdir -p /go/pkg /home/vscode/.cache \
    && chown vscode:golang /go/pkg \
    && chmod 2775 /go/pkg \
    && chown vscode:vscode /home/vscode/.cache

# //////////////////////////////////////////////////////////////////////////////////////////////////////////////////// #
