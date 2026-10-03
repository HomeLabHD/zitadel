# syntax=docker/dockerfile:1
# Self-contained zitadel backend build for the HomeLabHD fork (StageFreight builds this as a
# plain docker image — no external .artifacts). Build context is the REPO ROOT. Runs the full
# nx codegen (buf Go+OpenAPI, statik/sass, assets) and the Angular console build (embedded via
# go:embed) before `go build`, so the forked AuthRequest.privateLabelingOrgId field (#10692) is
# regenerated and compiled in. StageFreight auto-injects VERSION/COMMIT/BUILD_DATE; there is no
# upstream `branch` ldflag, so BRANCH is intentionally not wired.
############################
# Build stage (Go + Node for codegen/console)
############################
FROM golang:1.25-bookworm AS build
ARG VERSION=dev
ARG COMMIT=unknown
ARG BUILD_DATE            # RFC3339 (cmd/build/info.go parses time.RFC3339, else warns + uses now)
ARG ZITADEL_GOFIPS140=off # "certified" for a FIPS build
ARG TARGETARCH

ENV COREPACK_ENABLE_DOWNLOAD_PROMPT=0 \
    ENABLE_EXPERIMENTAL_COREPACK=1 \
    NX_DAEMON=false NX_NO_CLOUD=true NX_DISABLE_DB=true CI=true

# Node 22 (CI version) + pnpm via corepack; curl/unzip/git/bash for the console proto-plugin downloads + sass.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git unzip bash \
 && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
 && apt-get install -y --no-install-recommends nodejs \
 && corepack enable \
 && rm -rf /var/lib/apt/lists/*

# Non-root runtime user, created here so the scratch final can inherit it via /etc/passwd.
RUN useradd -s "" --home / zitadel

WORKDIR /src
COPY . .
RUN corepack prepare --activate && pnpm install --frozen-lockfile

# Full codegen (buf Go+OpenAPI, statik+sass, assets) + Angular console (embedded via go:embed).
# nx serialized (--parallel=1) so this fits a modest runner — zitadel's own CI uses 16-vCPU
# Depot runners with no limits; a smaller runner just trades speed for memory by not running
# the heavy tasks concurrently. The Angular console build wants a LARGE JS heap, so raise it
# for that step (too small a cap is itself a JS-heap OOM); codegen doesn't need it.
RUN pnpm nx run @zitadel/api:generate --parallel=1
RUN NODE_OPTIONS=--max-old-space-size=6144 pnpm nx run @zitadel/api:build-console --parallel=1

# Version-stamped static binary (ldflags from cmd/build/info.go).
RUN CGO_ENABLED=0 GOOS=linux GOARCH="${TARGETARCH}" GOFIPS140="${ZITADEL_GOFIPS140}" \
    go build -trimpath -o /zitadel \
      -ldflags="-s -w \
        -X github.com/zitadel/zitadel/cmd/build.version=${VERSION} \
        -X github.com/zitadel/zitadel/cmd/build.commit=${COMMIT} \
        -X github.com/zitadel/zitadel/cmd/build.date=${BUILD_DATE}"

############################
# Final minimal image (scratch) — mirrors upstream final stage
############################
FROM scratch AS final
COPY --from=build /etc/passwd /etc/passwd
COPY --from=build /etc/ssl/certs /etc/ssl/certs
COPY --from=build --chown=zitadel /zitadel /app/zitadel
HEALTHCHECK NONE
EXPOSE 8080
USER zitadel
ENTRYPOINT ["/app/zitadel"]
