# apps/api/Dockerfile
# ========================================
# Stage 1: Base
# ========================================
FROM node:20-alpine AS base

RUN corepack enable && corepack prepare pnpm@8.15.0 --activate
RUN apk add --no-cache libc6-compat

WORKDIR /app

# ========================================
# Stage 2: Pruner
# ========================================
FROM base AS pruner

RUN pnpm add -g turbo

COPY . .
RUN turbo prune --scope=api --docker

# ========================================
# Stage 3: Installer
# ========================================
FROM base AS installer

WORKDIR /app

COPY --from=pruner /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --from=pruner /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --from=pruner /app/out/json/ .

RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prefer-offline

COPY --from=pruner /app/out/full/ .

# ========================================
# Stage 4: Builder
# ========================================
FROM installer AS builder

WORKDIR /app

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION

ENV NODE_ENV=production

# Сборка API и всех зависимостей
RUN pnpm turbo run build --filter=api

# ========================================
# Stage 5: Production
# ========================================
FROM node:20-alpine AS production

RUN corepack enable && corepack prepare pnpm@8.15.0 --activate

RUN apk add --no-cache \
    tini \
    curl \
    ca-certificates \
    dumb-init

RUN addgroup -g 1001 -S nodejs && \
    adduser -u 1001 -S apiuser -G nodejs

WORKDIR /app

# Копирование артефактов
COPY --from=builder --chown=apiuser:nodejs /app/apps/api/dist ./apps/api/dist
COPY --from=builder --chown=apiuser:nodejs /app/apps/api/package.json ./apps/api/
COPY --from=builder --chown=apiuser:nodejs /app/packages ./packages
COPY --from=builder --chown=apiuser:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=apiuser:nodejs /app/package.json ./

USER apiuser

ENV NODE_ENV=production
ENV PORT=4000

EXPOSE 4000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:4000/health || exit 1

ENTRYPOINT ["/sbin/tini", "--"]

CMD ["node", "apps/api/dist/index.js"]
