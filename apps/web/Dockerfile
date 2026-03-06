# apps/web/Dockerfile
# ========================================
# Stage 1: Base - Установка pnpm
# ========================================
FROM node:20-alpine AS base

# Установка pnpm глобально
RUN corepack enable && corepack prepare pnpm@8.15.0 --activate

# Установка системных зависимостей
RUN apk add --no-cache libc6-compat

WORKDIR /app

# ========================================
# Stage 2: Pruner - Извлечение нужных пакетов
# ========================================
FROM base AS pruner

# Установка turbo для pruning
RUN pnpm add -g turbo

# Копирование всего репозитория
COPY . .

# Изолирование workspace для web приложения
RUN turbo prune --scope=web --docker

# ========================================
# Stage 3: Installer - Установка зависимостей
# ========================================
FROM base AS installer

WORKDIR /app

# Копирование lock файла и workspace конфигурации
COPY --from=pruner /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --from=pruner /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --from=pruner /app/out/json/ .

# Установка зависимостей с использованием кеша
RUN --mount=type=cache,id=pnpm,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile --prefer-offline

# Копирование исходного кода
COPY --from=pruner /app/out/full/ .

# ========================================
# Stage 4: Builder - Сборка приложения
# ========================================
FROM installer AS builder

WORKDIR /app

# Аргументы сборки
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
ARG NEXT_PUBLIC_API_URL
ARG NEXT_PUBLIC_APP_URL

ENV NEXT_TELEMETRY_DISABLED=1
ENV NODE_ENV=production

# Сборка с Turborepo
RUN pnpm turbo run build --filter=web

# ========================================
# Stage 5: Linter - Проверка качества кода
# ========================================
FROM installer AS linter

WORKDIR /app

# Запуск линтеров
RUN pnpm turbo run lint --filter=web
RUN pnpm turbo run type-check --filter=web

# ========================================
# Stage 6: Tester - Запуск тестов
# ========================================
FROM installer AS tester

WORKDIR /app

# Запуск тестов
RUN pnpm turbo run test --filter=web -- --passWithNoTests

# ========================================
# Stage 7: Production - Финальный образ
# ========================================
FROM node:20-alpine AS production

# Установка pnpm
RUN corepack enable && corepack prepare pnpm@8.15.0 --activate

# Установка runtime зависимостей
RUN apk add --no-cache \
    tini \
    curl \
    ca-certificates \
    tzdata

# Создание непривилегированного пользователя
RUN addgroup -g 1001 -S nodejs && \
    adduser -u 1001 -S nextjs -G nodejs

WORKDIR /app

# Метаданные
LABEL maintainer="devops@company.com" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${VCS_REF}" \
      org.opencontainers.image.version="${VERSION}"

# Копирование только необходимых файлов из builder
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/web/public ./apps/web/public

# Переключение на непривилегированного пользователя
USER nextjs

# Переменные окружения
ENV NODE_ENV=production
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:3000/api/health || exit 1

# Использование Tini для правильной обработки сигналов
ENTRYPOINT ["/sbin/tini", "--"]

# Запуск приложения
CMD ["node", "apps/web/server.js"]
