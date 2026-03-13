# This dockerfile builds an image for the backend package.
# It should be executed with the root of the repo as docker context.
#
# Before building this image, be sure to have run the following commands in the repo root:
#
# yarn install --immutable
# yarn tsc
# yarn build:backend
#
# Once the commands have been run, you can build the image using `yarn build-image`

# Stage 1: builder — instala dependências e compila módulos nativos
FROM node:24-trixie-slim AS builder

# Set Python interpreter for `node-gyp` to use
ENV PYTHON=/usr/bin/python3

# Install build tools for native modules (isolated-vm, better-sqlite3)
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends python3 g++ build-essential libsqlite3-dev && \
    rm -rf /var/lib/apt/lists/*

USER node
WORKDIR /app

# Copy files needed by Yarn
COPY --chown=node:node .yarn ./.yarn
COPY --chown=node:node .yarnrc.yml ./
COPY --chown=node:node backstage.json ./

ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

COPY --chown=node:node yarn.lock package.json packages/backend/dist/skeleton.tar.gz ./
RUN tar xzf skeleton.tar.gz && rm skeleton.tar.gz

RUN --mount=type=cache,target=/home/node/.cache/yarn,sharing=locked,uid=1000,gid=1000 \
    yarn workspaces focus --all --production && rm -rf "$(yarn cache clean)"

COPY --chown=node:node examples ./examples
COPY --chown=node:node packages/backend/dist/bundle.tar.gz app-config*.yaml ./
RUN tar xzf bundle.tar.gz && rm bundle.tar.gz

# Stage 2: runtime — distroless, zero build tools, zero shell
FROM gcr.io/distroless/nodejs24-debian13:nonroot

ENV NODE_ENV=production
ENV NODE_OPTIONS="--no-node-snapshot"

WORKDIR /app
COPY --from=builder --chown=nonroot:nonroot /app /app
