# syntax=docker/dockerfile:1
# Self-contained login-v2 build for the HomeLabHD fork (StageFreight builds this as a plain
# docker image — no external .artifacts). Build context is the REPO ROOT (needs proto/,
# packages/, root lockfile), not apps/login. Regenerates the TS proto bindings so the login
# compiles against the forked AuthRequest.privateLabelingOrgId field (#10692 fix).
############################
# Build stage
############################
FROM node:22-bookworm AS build
ENV COREPACK_ENABLE_DOWNLOAD_PROMPT=0 \
    ENABLE_EXPERIMENTAL_COREPACK=1 \
    NX_DAEMON=false NX_NO_CLOUD=true NX_DISABLE_DB=true CI=true
RUN corepack enable
WORKDIR /src
COPY . .
RUN corepack prepare --activate && pnpm install --frozen-lockfile
# proto:generate (TS bindings) -> client build -> login standalone (nx chain also pulls these,
# but proto:generate is run explicitly so the .proto change is regenerated deterministically).
RUN pnpm nx run @zitadel/proto:generate
RUN pnpm nx run @zitadel/login:build

############################
# Runtime (mirrors upstream apps/login/Dockerfile)
############################
FROM node:24-alpine AS final
WORKDIR /app
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
RUN mkdir -p /.env-file && touch /.env-file/.env && chown -R nextjs:nodejs /.env-file
COPY --from=build --chown=nextjs:nodejs /src/apps/login/.next/standalone ./
USER nextjs
ENV HOSTNAME="::" PORT="3000" NODE_ENV="production" \
    NODE_OPTIONS="--use-openssl-ca --require /app/load-ssl-cert-dir.cjs" \
    SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt" \
    ZITADEL_TLS_ENABLED="false" \
    OTEL_SERVICE_NAME="zitadel-login" \
    OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD ["/usr/local/bin/node", "/app/healthcheck.mjs", "/ui/v2/login/ready"]
ENTRYPOINT ["/app/entrypoint.sh", "node", "apps/login/server.js"]
