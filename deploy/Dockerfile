# syntax=docker/dockerfile:1.7
FROM node:24-trixie AS build

ENV CI=true \
    DSH_TELEMETRY_DISABLED=1 \
    ELECTRON_SKIP_BINARY_DOWNLOAD=1 \
    PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    DSH_GATE_CONCURRENCY=2

RUN apt-get update \
    && apt-get install -y --no-install-recommends musl-tools \
    && rm -rf /var/lib/apt/lists/*

ARG VCS_REF
ENV DSH_CLIENT_COMMIT_HASH=${VCS_REF}
WORKDIR /source
COPY . .
RUN corepack enable \
    && pnpm install --frozen-lockfile \
    && pnpm run build:official \
    && pnpm --dir native/system run build:ts \
    && pnpm --dir native/system run build:native \
    && sed -i "s|'link:vendor/|'file:vendor/|g" pnpm-workspace.yaml \
    && pnpm --filter dsh-python-runtime-closure deploy --prod --legacy /opt/dsh

FROM node:24-trixie-slim AS runtime

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
      ca-certificates curl git openssh-client python3 ripgrep tini \
    && rm -rf /var/lib/apt/lists/* \
    && npm install --global --no-audit --no-fund pnpm@11.28.5

COPY --from=build /opt/dsh /opt/dsh
COPY deploy/entrypoint.sh /usr/local/bin/dsh-container
RUN chmod 0755 /usr/local/bin/dsh-container \
    && chmod 0755 /opt/dsh/node_modules/@deepseek-ai/dsh/lib/bin.js \
    && ln -s /opt/dsh/node_modules/@deepseek-ai/dsh/lib/bin.js /usr/local/bin/dsh \
    && mkdir -p /home/node/.dsh /home/node/.agents /workspace \
    && chown node:node /home/node/.dsh /home/node/.agents /workspace

ARG VCS_REF
LABEL org.opencontainers.image.source="https://github.com/Gorvey/deepseek-harness" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.licenses="MIT"

ENV NODE_ENV=production \
    DSH_HOME=/home/node/.dsh \
    DSH_AGENTS_HOME=/home/node/.agents \
    DSH_TELEMETRY_DISABLED=1

USER node
RUN test "$(pnpm --version)" = 11.28.5 \
    && node /opt/dsh/node_modules/@deepseek-ai/dsh/lib/bin.js --version
WORKDIR /workspace
EXPOSE 3080
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/dsh-container"]
CMD ["web", "--no-open"]
