---
title: "Docker — исчерпывающая шпаргалка"
version: "2.0.0"
last_updated: "2026-03-13"
author: "Senior DevOps Engineer"
---

# 🐳 Docker — Production Cheatsheet

> **Аудитория:** Инженеры с опытом Linux и базовым знанием Docker.  
> **Docker Engine:** 25+ / Compose V2 / BuildKit по умолчанию.

---

## Содержание

1. [Ключевые концепции](#1-ключевые-концепции)
2. [Основные команды CLI](#2-основные-команды-cli)
3. [Лучшие практики Dockerfile](#3-лучшие-практики-dockerfile)
4. [Сетевое взаимодействие](#4-сетевое-взаимодействие)
5. [Тома и хранилище](#5-тома-и-хранилище)
6. [Docker Compose](#6-docker-compose)
7. [Харденинг безопасности](#7-харденинг-безопасности)
8. [Производительность и управление ресурсами](#8-производительность-и-управление-ресурсами)
9. [Registry и управление образами](#9-registry-и-управление-образами)
10. [Production-паттерны](#10-production-паттерны)
11. [Быстрый справочник](#быстрый-справочник)
12. [Дополнительные материалы](#дополнительные-материалы)

---

## 1. Ключевые концепции

Docker строится на изоляции ядра Linux: **namespaces** (pid, net, mnt, uts, ipc, user) дают каждому контейнеру изолированное представление системных ресурсов, а **cgroups v2** ограничивают их потребление. Образы — это read-only стек слоёв на базе **OverlayFS** (union filesystem); контейнер добавляет поверх тонкий writable слой (copy-on-write).

```
IMAGE LAYER STRUCTURE
─────────────────────────────────────
  [R/W] Container layer  ← diff, COW
  [R/O] App layer        ← COPY . /app
  [R/O] Deps layer       ← RUN pip install
  [R/O] Base layer       ← FROM python:3.12-slim
─────────────────────────────────────
Shared between all containers of same image
```

```bash
# Просмотр истории слоёв и их размеров
docker image history --no-trunc nginx:1.27-alpine

# Инспектировать union-mount точки живого контейнера
docker inspect <cid> | jq '.[0].GraphDriver.Data'

# Подтвердить namespace-изоляцию: PID 1 внутри != снаружи
docker run --rm alpine sh -c 'echo $$'       # → 1
echo $$                                       # → 12345+

# cgroups v2: увидеть иерархию ресурсов контейнера
systemctl status docker
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect <cid> --format '{{.Id}}').scope/memory.current
```

> ⚠️ **Антипаттерн:** Хранить изменяемые данные в writable слое контейнера. COW-операции дорогостоящи при интенсивной записи; данные теряются при `docker rm`. Используйте тома.

> ✅ **Senior-совет:** На ядрах с cgroups v1 `--memory-swap` имеет иную семантику, чем v2. Проверяйте: `docker info | grep "Cgroup Version"`. На v2 `--memory-swap` задаёт swap сверх RAM, на v1 — суммарный лимит RAM+swap.

---

## 2. Основные команды CLI

Флаги `docker run` — наиболее критичная область; неправильное использование ведёт к утечкам ресурсов и дырам безопасности. Приведённые примеры — готовые production-шаблоны.

```bash
# ── Жизненный цикл контейнера ──────────────────────────────────────────────

# Запустить с полным набором production-флагов
docker run -d \
  --name api \
  --restart unless-stopped \
  --memory 512m --memory-swap 512m \
  --cpus 1.5 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  -e NODE_ENV=production \
  --env-file .env.prod \
  -p 127.0.0.1:3000:3000 \
  --health-cmd "curl -fs http://localhost:3000/healthz || exit 1" \
  --health-interval 15s \
  --health-retries 3 \
  myapp:1.4.2@sha256:<digest>

# Graceful stop с таймаутом (даёт время SIGTERM-обработчику)
docker stop --time 30 api

# ── Отладка ─────────────────────────────────────────────────────────────────

# Exec в работающий контейнер (без bash — используем sh)
docker exec -it api sh

# Просмотр живых логов с таймштампами, последние 100 строк
docker logs --tail 100 --timestamps --follow api

# Статистика ресурсов без потока (одноразовый снимок)
docker stats --no-stream --format \
  "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}"

# Diff writable слоя (что изменилось в FS)
docker diff api

# Копировать файл из/в контейнер без exec
docker cp api:/app/config.yaml ./config.yaml
docker cp ./config.yaml api:/app/config.yaml

# ── Образы ──────────────────────────────────────────────────────────────────

# Сборка с BuildKit, кэшем и явной платформой
DOCKER_BUILDKIT=1 docker build \
  --platform linux/amd64 \
  --cache-from type=registry,ref=registry.example.com/myapp:buildcache \
  --cache-to   type=registry,ref=registry.example.com/myapp:buildcache,mode=max \
  --build-arg VERSION=$(git describe --tags --dirty) \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t myapp:$(git rev-parse --short HEAD) \
  -f docker/Dockerfile .

# Скопировать образ между registry без pull/push через daemon
docker buildx imagetools create \
  --tag registry2.example.com/myapp:1.4.2 \
  registry1.example.com/myapp:1.4.2

# Inspect multi-arch манифеста
docker buildx imagetools inspect nginx:1.27-alpine

# Удалить dangling-образы (не только untagged, но и unused)
docker image prune --filter "until=240h"

# ── Системная очистка ────────────────────────────────────────────────────────

# Агрессивная очистка (НЕ запускать на prod-хосте без осмотра)
docker system prune --volumes --filter "until=720h"

# Посмотреть занимаемое место до прунинга
docker system df -v
```

> ⚠️ **Антипаттерн:** `docker run ... --restart always` на контейнерах без healthcheck. При баге в старте контейнер войдёт в бесконечный crash-loop, дестабилизируя хост. Всегда комбинируйте с `--health-*` или управляйте жизненным циклом через оркестратор.

> ✅ **Senior-совет:** `docker exec` открывает shell внутри namespaces контейнера, но **не** создаёт новый cgroup. Это значит, что CPU/memory лимиты родительского контейнера применяются к exec-сессии. При профилировании из exec помните: вы «внутри» тех же ограничений.

---

## 3. Лучшие практики Dockerfile

Порядок инструкций критичен для кэширования слоёв: редко меняющиеся инструкции — вверху, часто меняющиеся — внизу. Многоэтапные сборки (multi-stage) позволяют оставить в финальном образе только бинарник и runtime-зависимости.

```dockerfile
# syntax=docker/dockerfile:1.7
# ─── Stage 1: зависимости (кэшируемый слой) ──────────────────────────────────
FROM node:22-bookworm-slim AS deps

WORKDIR /app

# Отдельный COPY для package*.json → кэш сброшен только при изменении lockfile
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev --ignore-scripts

# ─── Stage 2: сборка ──────────────────────────────────────────────────────────
FROM node:22-bookworm-slim AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --ignore-scripts

COPY . .
RUN npm run build

# ─── Stage 3: финальный образ ─────────────────────────────────────────────────
FROM node:22-bookworm-slim AS runtime

# OCI Labels (стандарт https://github.com/opencontainers/image-spec)
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.licenses="MIT"

# Установка только runtime-зависимостей; --no-install-recommends экономит ~30%
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        tini \
    && rm -rf /var/lib/apt/lists/*

# Создаём non-root пользователя без shell
RUN groupadd --system --gid 1001 appgroup && \
    useradd  --system --uid 1001 --gid appgroup --no-create-home --shell /sbin/nologin appuser

WORKDIR /app

# Копируем только необходимое из предыдущих стадий
COPY --from=deps     --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder  --chown=appuser:appgroup /app/dist         ./dist
COPY --chown=appuser:appgroup package.json ./

# ARG — только build-time; ENV — доступна в runtime
ARG VERSION=dev
ARG BUILD_DATE
ENV NODE_ENV=production \
    PORT=3000 \
    APP_VERSION=${VERSION}

USER appuser

EXPOSE 3000

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
    CMD curl -fs http://localhost:${PORT}/healthz || exit 1

# tini как init-процесс: корректная передача сигналов, zombie reaping
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "dist/server.js"]
```

```dockerfile
# ─── Go multi-stage — итоговый образ 0 зависимостей ──────────────────────────
# syntax=docker/dockerfile:1.7
FROM golang:1.22-bookworm AS builder

WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w -extldflags=-static" \
    -o /out/app ./cmd/app

FROM scratch AS runtime
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /out/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
```

```
# .dockerignore — критично для контекста и безопасности
**/.git
**/.gitignore
**/.env*
**/node_modules
**/dist
**/coverage
**/__pycache__
**/*.pyc
**/Dockerfile*
**/docker-compose*.yml
**/.dockerignore
**/README.md
**/*.test.*
**/*.spec.*
**/secrets/
```

> ⚠️ **Антипаттерн:** `COPY . .` в первых строках Dockerfile. Любое изменение любого файла проекта (включая README.md) инвалидирует кэш всех последующих слоёв. Всегда копируйте манифесты зависимостей первыми, устанавливайте зависимости, затем копируйте код.

> ✅ **Senior-совет:** `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci` позволяет использовать секреты во время сборки без записи их в слой образа. Секрет не остаётся в `docker history`. Это единственный правильный способ использовать `.npmrc`, токены или `.netrc` при сборке.

---

## 4. Сетевое взаимодействие

Docker поддерживает несколько сетевых драйверов; для production микросервисов используйте user-defined bridge для единичного хоста или overlay для Swarm-кластера. DNS-резолюция по имени сервиса работает только в user-defined сетях, но не в `bridge` по умолчанию.

```bash
# ── Инспекция и управление сетями ───────────────────────────────────────────

# Создать изолированную сеть с кастомным диапазоном
docker network create \
  --driver bridge \
  --subnet 172.20.0.0/16 \
  --ip-range 172.20.240.0/20 \
  --gateway 172.20.0.1 \
  --opt com.docker.network.bridge.name=docker-prod \
  prod-net

# Подключить работающий контейнер к дополнительной сети
docker network connect --alias api-v2 prod-net my-api

# DNS-резолюция между контейнерами в одной user-defined сети
docker run --rm --network prod-net alpine \
  nslookup api    # → резолвится через embedded DNS (127.0.0.11)

# Отладка сети без установки инструментов в целевой контейнер
docker run --rm --network container:my-api \
  nicolaka/netshoot \
  ss -tlnp

# Просмотр iptables-правил, созданных Docker
iptables -t nat -L DOCKER --line-numbers -v

# Bind только на localhost: безопаснее, чем 0.0.0.0
docker run -p 127.0.0.1:5432:5432 postgres:16-alpine

# macvlan — контейнер с реальным MAC/IP в LAN-сети
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  macvlan-net

# overlay для Swarm (multi-host)
docker network create \
  --driver overlay \
  --attachable \
  --opt encrypted=true \
  swarm-net
```

```yaml
# docker-compose: явная изоляция сетей по слоям
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true   # нет доступа в интернет
  monitoring:
    driver: bridge

services:
  nginx:
    networks: [frontend, backend]

  api:
    networks: [backend, monitoring]

  db:
    networks: [backend]   # только internal
```

> ⚠️ **Антипаттерн:** `--network host` в production-контейнерах. Контейнер полностью разделяет сетевой стек хоста: нет изоляции портов, любой процесс в контейнере видит весь трафик интерфейсов. Допустимо только для мониторинговых агентов (node-exporter) или сетевых инструментов.

> ✅ **Senior-совет:** Embedded DNS (127.0.0.11) в Docker поддерживает **поиск по алиасам** (`--alias`), а не только по имени контейнера. Это позволяет реализовать blue/green внутри compose: подключить новую версию сервиса под тем же алиасом до отключения старой, обеспечив нулевое downtime-переключение без внешнего балансировщика.

---

## 5. Тома и хранилище

Три типа хранилища с принципиально разными характеристиками: **named volumes** — под управлением Docker, оптимальны для БД; **bind mounts** — прямой доступ к хост-FS, для разработки и конфигов; **tmpfs** — только в памяти, для временных данных и секретов.

```bash
# ── Named volumes ────────────────────────────────────────────────────────────

# Создать volume с явным драйвером
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw,nfsvers=4 \
  --opt device=:/exports/data \
  nfs-data

# Backup тома через временный контейнер (паттерн sidecar-backup)
docker run --rm \
  -v pgdata:/source:ro \
  -v $(pwd)/backups:/backup \
  alpine \
  tar czf /backup/pgdata-$(date +%Y%m%d_%H%M%S).tar.gz -C /source .

# Restore
docker run --rm \
  -v pgdata:/target \
  -v $(pwd)/backups:/backup:ro \
  alpine \
  tar xzf /backup/pgdata-20240101_120000.tar.gz -C /target

# Клонировать volume
docker run --rm \
  -v pgdata-prod:/from:ro \
  -v pgdata-staging:/to \
  alpine \
  sh -c "cd /from && cp -a . /to/"

# Просмотр usage всех volumes
docker system df -v | grep -A 100 "VOLUME NAME"

# ── Bind mounts ──────────────────────────────────────────────────────────────

# Read-only bind mount конфига
docker run -v /etc/myapp/config.yaml:/app/config.yaml:ro,Z myapp:latest
# :Z — пересоздать SELinux label для private unshared

# ── tmpfs ────────────────────────────────────────────────────────────────────

# tmpfs с ограничением: не exec, не setuid, 128MB
docker run \
  --tmpfs /run:rw,noexec,nosuid,size=128m \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  myapp:latest
```

```yaml
# Compose: volumes с explicit driver и backup labels
volumes:
  pgdata:
    driver: local
    labels:
      backup.enabled: "true"
      backup.schedule: "0 2 * * *"
  redis-data:
    driver: local

services:
  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
      - /etc/pg/postgresql.conf:/etc/postgresql/postgresql.conf:ro

  api:
    volumes:
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 67108864  # 64MB в байтах
```

> ⚠️ **Антипаттерн:** Bind-mounting всей директории проекта в production-образ (`-v /app:/app`). Это отменяет всё, что было скопировано в образ на этапе сборки, включая скомпилированные артефакты. Bind mounts только для конфигов, сертификатов и в dev-окружении.

> ✅ **Senior-совет:** `docker volume inspect <vol> | jq '.[0].Mountpoint'` покажет реальный путь на хосте. На systemd-хостах это `/var/lib/docker/volumes/<name>/_data`. Backup volumes можно делать **snapshot'ами LVM/ZFS** на уровне хоста — быстрее и атомарнее, чем tar через контейнер, особенно для больших БД.

---

## 6. Docker Compose

Compose V2 (`docker compose`) — встроенный плагин Docker Engine 25+. Декларативное описание сервисов, сетей и томов. Используйте `depends_on` с `condition: service_healthy` вместо `service_started` для корректного порядка запуска.

```yaml
# docker-compose.yml — production-ready шаблон
name: myapp

x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

x-restart: &default-restart
  restart: unless-stopped

services:
  postgres:
    image: postgres:16-alpine
    <<: *default-restart
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks: [backend]
    logging: *default-logging
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: 1G

  redis:
    image: redis:7-alpine
    <<: *default-restart
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASSWORD}"]
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks: [backend]
    logging: *default-logging

  api:
    image: myapp:${IMAGE_TAG:-latest}
    <<: *default-restart
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://${DB_USER}@postgres:5432/${DB_NAME}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379
    env_file:
      - .env
      - .env.local   # override, gitignored
    secrets:
      - db_password
    ports:
      - "127.0.0.1:3000:3000"
    networks: [frontend, backend]
    read_only: true
    tmpfs:
      - /tmp:exec,nosuid
    security_opt:
      - no-new-privileges:true
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    logging: *default-logging
    healthcheck:
      test: ["CMD", "curl", "-fs", "http://localhost:3000/healthz"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s

  nginx:
    image: nginx:1.27-alpine
    <<: *default-restart
    depends_on:
      api:
        condition: service_healthy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks: [frontend]
    logging: *default-logging

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  pgdata:
  redis-data:

networks:
  frontend:
  backend:
    internal: true
```

```yaml
# docker-compose.override.yml — автоматически применяется в dev
# НЕ коммитить если содержит секреты
services:
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: builder    # использовать stage с devtools
    volumes:
      - .:/app
      - /app/node_modules   # anonymous volume, не перезаписывать
    environment:
      NODE_ENV: development
      DEBUG: "app:*"
    ports:
      - "9229:9229"   # Node.js debugger
```

```yaml
# docker-compose.prod.yml — явный override для CI/CD
# docker compose -f docker-compose.yml -f docker-compose.prod.yml up
services:
  api:
    image: registry.example.com/myapp:${CI_COMMIT_SHA}
    environment:
      NODE_ENV: production
    deploy:
      replicas: 3
```

```bash
# Profiles: запускать только нужные сервисы
# docker compose --profile debug up
services:
  pgadmin:
    image: dpage/pgadmin4
    profiles: ["debug", "admin"]

# Управление
docker compose up -d --wait              # ждать healthy всех сервисов
docker compose up -d --scale api=3       # горизонтальное масштабирование
docker compose logs -f --tail 50 api     # логи конкретного сервиса
docker compose exec api sh               # exec внутрь
docker compose ps -a --format json       # статус в JSON
docker compose config --resolve-image-digests  # финальный resolved config
```

> ⚠️ **Антипаттерн:** `depends_on: [postgres]` без `condition`. По умолчанию это `condition: service_started` — контейнер просто запущен, но БД ещё инициализируется. Приложение упадёт с ошибкой подключения. Без healthcheck в сервисе-зависимости `service_healthy` не работает.

> ✅ **Senior-совет:** YAML-анкоры (`&default-logging`, `<<: *default-logging`) — нативный способ DRY в Compose без extension fields. Но будьте осторожны: `<<:` (merge key) — это YAML 1.1 фича, не поддерживаемая строгим YAML 1.2. Docker Compose использует yaml.v3 (Go), который поддерживает merge keys. Однако в Compose V2 появились нативные `x-` extension fields с `&anchors` — используйте их для более явного кода.

---

## 7. Харденинг безопасности

Контейнеры по умолчанию запускаются с избыточными привилегиями. Defense-in-depth требует многоуровневого подхода: минимальный образ, non-root пользователь, ограниченные capabilities, seccomp-профиль, read-only FS и регулярное сканирование.

```bash
# ── Анализ и сканирование ────────────────────────────────────────────────────

# Trivy: сканирование образа на CVE + мисконфигурации Dockerfile
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/Library/Caches:/root/.cache/ \
  aquasec/trivy:latest image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --format table \
  myapp:latest

# Генерация SBOM в CycloneDX формате
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  anchore/syft:latest \
  packages docker:myapp:latest \
  -o cyclonedx-json > sbom.json

# Проверка образа на CIS Docker Benchmark
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  docker/docker-bench-security

# ── Runtime ограничения ──────────────────────────────────────────────────────

# Минимальный набор capabilities (DROP ALL, добавить только нужное)
docker run \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --security-opt seccomp=/etc/docker/seccomp/default.json \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --user 1001:1001 \
  myapp:latest

# AppArmor профиль (Ubuntu/Debian)
docker run \
  --security-opt apparmor=docker-default \
  myapp:latest

# ── Секреты ──────────────────────────────────────────────────────────────────

# НЕ так:
# docker run -e DB_PASSWORD=mysecret ...

# Docker secrets (Swarm или Compose)
echo "mysecretpassword" | docker secret create db_password -
docker service create \
  --secret db_password \
  --env DB_PASSWORD_FILE=/run/secrets/db_password \
  myapp:latest

# BuildKit secret mount (не попадает в историю слоёв)
docker build \
  --secret id=npmrc,src=$HOME/.npmrc \
  .

# Dockerfile usage:
# RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

```bash
# ── Проверка текущих capabilities контейнера ────────────────────────────────
docker inspect <cid> | jq '.[0].HostConfig | {CapAdd, CapDrop, SecurityOpt, ReadonlyRootfs}'

# ── Изоляция через user namespaces ──────────────────────────────────────────
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}
# UID 0 в контейнере → непривилегированный UID на хосте
```

```dockerfile
# Минимальный образ: distroless (no shell = меньше attack surface)
FROM gcr.io/distroless/nodejs22-debian12:nonroot AS runtime
COPY --from=builder /app/dist /app
EXPOSE 3000
CMD ["/app/server.js"]
```

> ⚠️ **Антипаттерн:** Монтирование `/var/run/docker.sock` в контейнеры приложений. Это эквивалентно root-доступу к хосту: любой процесс внутри может запустить `docker run --privileged`. Для CI/CD используйте `docker-in-docker` (dind) или rootless buildkit (`buildkitd`).

> ✅ **Senior-совет:** `--security-opt no-new-privileges:true` блокирует `setuid`/`setgid` битовые программы от получения привилегий (прим.: `sudo`, `su`, `passwd`). Этот флаг дешевле CAP_DROP ALL, потому что работает через `prctl(PR_SET_NO_NEW_PRIVS)` без модификации capabilities-маски. В контейнерах без shell он, тем не менее, критичен — защищает от эксплойтов через suid-бинари базового образа.

---

## 8. Производительность и управление ресурсами

cgroups ограничивают ресурсы на уровне ядра. Без лимитов один контейнер может исчерпать ресурсы хоста, вызвав OOM-killer или CPU starvation. В production **обязательно** задавать `--memory` и `--cpus`.

```bash
# ── CPU ─────────────────────────────────────────────────────────────────────

# --cpus: мягкое ограничение (throttling при превышении)
docker run --cpus 1.5 myapp:latest         # 1.5 физических ядра

# --cpu-period + --cpu-quota: явная квота (1.5 = 150000/100000)
docker run --cpu-period 100000 --cpu-quota 150000 myapp:latest

# --cpu-shares: относительный вес (default 1024)
docker run --cpu-shares 512 low-prio:latest   # половина default
docker run --cpu-shares 2048 high-prio:latest # двойной приоритет

# Прибить к конкретным NUMA/CPU ядрам
docker run --cpuset-cpus "0,1" myapp:latest

# ── Memory ───────────────────────────────────────────────────────────────────

# --memory: hard limit; --memory-reservation: soft limit (scheduling hint)
docker run \
  --memory 512m \
  --memory-swap 512m \         # swap = 0 (swap = total - memory; 512-512=0)
  --memory-reservation 256m \  # 256m soft hint
  --oom-score-adj 500 \        # повысить вероятность OOM-kill этого контейнера
  myapp:latest

# Мониторинг OOM событий в реальном времени
docker events --filter event=oom

# ── Ulimits ──────────────────────────────────────────────────────────────────

docker run \
  --ulimit nofile=65536:65536 \   # open file descriptors
  --ulimit nproc=4096:4096 \      # max processes
  --ulimit core=0 \               # no core dumps
  myapp:latest

# ── Логирование ──────────────────────────────────────────────────────────────

# json-file с ротацией (default driver)
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=5 \
  --log-opt compress=true \
  myapp:latest

# Отправка в Loki
docker run \
  --log-driver loki \
  --log-opt loki-url="http://loki:3100/loki/api/v1/push" \
  --log-opt loki-batch-size=400 \
  --log-opt loki-timeout=2s \
  --log-opt keep-file=true \
  myapp:latest

# Fluentd/Fluentbit (структурированные логи)
docker run \
  --log-driver fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="docker.{{.Name}}" \
  myapp:latest
```

```bash
# ── Профилирование в реальном времени ────────────────────────────────────────

# Просмотр cgroup метрик напрямую
CID=$(docker inspect myapp --format '{{.Id}}')
cat /sys/fs/cgroup/memory/docker/${CID}/memory.stat
cat /sys/fs/cgroup/cpu/docker/${CID}/cpu.stat

# Benchmark: измерить CPU throttling
docker stats --no-stream myapp | awk '{print $3}'  # CPU %

# pprof через nsenter без установки инструментов в контейнер
PID=$(docker inspect myapp --format '{{.State.Pid}}')
nsenter -t $PID -n curl http://localhost:6060/debug/pprof/heap > heap.pb.gz
```

> ⚠️ **Антипаттерн:** `--memory-swap -1` (unlimited swap). При OOM контейнер начинает активно использовать swap, деградируя до неприемлемо медленной скорости вместо завершения. Лучше явный crash с алертом, чем зависший контейнер, трёт swap часами.

> ✅ **Senior-совет:** `--memory-swap` = RAM + Swap суммарно на cgroups v1, но на cgroups v2 (`memory.swap.max`) — это только swap сверх RAM. Проверяйте версию: `docker info | grep "Cgroup Version"`. Неправильная интерпретация ведёт к вдвое меньшему лимиту, чем ожидалось, или к неожиданному OOM.

---

## 9. Registry и управление образами

Правильная стратегия тегирования — основа reproducibility. Никогда не деплоить `:latest` в production: тег мутабелен. Закрепляйте по digest (`@sha256:...`) для детерминированных развёртываний.

```bash
# ── Тегирование ──────────────────────────────────────────────────────────────

# Семантическое тегирование: version + git SHA + date
IMAGE_TAG="1.4.2"
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +%Y%m%d)

docker tag myapp:build \
  registry.example.com/myapp:${IMAGE_TAG}
docker tag myapp:build \
  registry.example.com/myapp:${IMAGE_TAG}-${GIT_SHA}
docker tag myapp:build \
  registry.example.com/myapp:${IMAGE_TAG}-${BUILD_DATE}

# Получить digest после push
DIGEST=$(docker push registry.example.com/myapp:${IMAGE_TAG} \
  | grep "digest:" | awk '{print $3}')

# Deploy по digest (immutable)
docker pull registry.example.com/myapp@${DIGEST}

# ── Multi-arch сборки с buildx ───────────────────────────────────────────────

# Создать builder с multi-platform support
docker buildx create \
  --name multiarch \
  --driver docker-container \
  --driver-opt network=host \
  --use

# Сборка и push multi-arch манифеста одной командой
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --push \
  --provenance=true \
  --sbom=true \
  --cache-from type=registry,ref=registry.example.com/myapp:buildcache \
  --cache-to   type=registry,ref=registry.example.com/myapp:buildcache,mode=max \
  -t registry.example.com/myapp:${IMAGE_TAG} \
  -t registry.example.com/myapp:latest \
  .

# Inspect манифест: убедиться что все архитектуры присутствуют
docker buildx imagetools inspect registry.example.com/myapp:${IMAGE_TAG}

# ── Оптимизация образов ──────────────────────────────────────────────────────

# Dive: анализ слоёв интерактивно
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive:latest \
  registry.example.com/myapp:${IMAGE_TAG}

# Скраппер неиспользованных образов без удаления работающих
docker image prune \
  --filter "dangling=true" \
  --filter "until=168h"   # старше 7 дней

# Найти образы без тегов (dangling)
docker images --filter "dangling=true" --format "{{.ID}}: {{.Size}}"

# Registry: удаление по тегу через API (Harbor/Distribution)
curl -X DELETE \
  -u user:password \
  "https://registry.example.com/v2/myapp/manifests/${DIGEST}"

# ── Signing (cosign) ─────────────────────────────────────────────────────────

# Подписать образ после push
cosign sign --key cosign.key registry.example.com/myapp@${DIGEST}

# Верифицировать подпись
cosign verify --key cosign.pub registry.example.com/myapp@${DIGEST}
```

> ⚠️ **Антипаттерн:** `FROM ubuntu:latest` или любой `FROM image:latest` в Dockerfile. Тег `latest` меняется при каждом релизе апстрима; ваша сборка непредсказуемо изменится. Всегда пинайте точную версию: `FROM ubuntu:24.04` или `FROM ubuntu@sha256:<digest>` для абсолютной воспроизводимости.

> ✅ **Senior-совет:** BuildKit `--cache-to mode=max` экспортирует **все** промежуточные слои в registry-кэш, не только финальные. При следующей сборке даже изменение в середине Dockerfile может переиспользовать кэш начала. `mode=min` (default) экспортирует только слои финальной стадии — дешевле хранить, но хуже попадание кэша при multi-stage.

---

## 10. Production-паттерны

Production-контейнер — это не просто `docker run`. Корректная обработка сигналов, healthcheck, init-процесс и паттерны observability превращают контейнер из разработческой игрушки в надёжный сервис.

```bash
# ── Graceful shutdown ────────────────────────────────────────────────────────

# Docker посылает SIGTERM → ждёт --stop-timeout → SIGKILL
# Приложение должно ловить SIGTERM и завершаться корректно

# Проверить поведение
docker stop --time 30 myapp   # 30 секунд на SIGTERM-обработку

# strace: убедиться что PID 1 получает сигнал
docker run --rm --pid=container:myapp \
  alpine sh -c 'kill -SIGTERM 1; sleep 1'
```

```bash
# ── Init-процесс: tini ───────────────────────────────────────────────────────

# Без init: PID 1 — это ваше приложение. Оно должно:
# 1. Правильно форвардить сигналы дочерним процессам
# 2. Собирать зомби-процессы (wait(2) для orphaned children)
# Большинство приложений этого не делают корректно.

# tini решает обе проблемы:
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "server.js"]

# Или через --init флаг Docker (встроенный tini):
docker run --init myapp:latest

# ── Healthcheck ──────────────────────────────────────────────────────────────

# Многоуровневый healthcheck: проверяем не просто /healthz но и зависимости
HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
    CMD curl -fs http://localhost:3000/healthz || exit 1

# /healthz endpoint должен проверять:
# - Само приложение (память, goroutine count)
# - Коннекции к БД (ping)
# - Коннекции к Redis
# НЕ должен проверять внешние сервисы (CI/CD замкнётся в зависимостях)
```

```bash
# ── Паттерн Sidecar ──────────────────────────────────────────────────────────

# Логирование: sidecar читает логи основного контейнера через shared volume
docker run -d --name app \
  -v app-logs:/var/log/app \
  myapp:latest

docker run -d --name log-shipper \
  -v app-logs:/var/log/app:ro \
  --volumes-from app \     # альтернатива shared volume
  fluent/fluent-bit:latest

# ── Отладка живых контейнеров ────────────────────────────────────────────────

# Inspect процессов без exec (через /proc)
PID=$(docker inspect myapp --format '{{.State.Pid}}')
ls -la /proc/$PID/fd | wc -l          # open file descriptors
cat /proc/$PID/net/tcp6               # network connections
cat /proc/$PID/status | grep VmRSS    # RSS memory

# nsenter: войти в namespaces без изменения cgroups (нет exec overhead)
nsenter -t $PID -n -u -i -p \
  -- ss -tlnp                         # network tools в namespace контейнера

# Добавить ephemeral debug-контейнер к работающему поду (kubectl-style в docker)
docker run --rm -it \
  --network container:myapp \
  --pid container:myapp \
  --volumes-from myapp \
  nicolaka/netshoot                   # полный набор сетевых инструментов

# Профилировать Go-приложение без restart
docker exec myapp kill -SIGUSR1 1     # если настроен SIGUSR1 handler
# Или через HTTP pprof:
docker exec myapp curl -s http://localhost:6060/debug/pprof/ 

# ── Обработка конфигурации ───────────────────────────────────────────────────

# Паттерн: envsubst для шаблонизации конфигов при старте
docker run --rm \
  -e DB_HOST=postgres \
  -e DB_PORT=5432 \
  -v $(pwd)/nginx.conf.template:/etc/nginx/templates/default.conf.template \
  nginx:1.27-alpine
# nginx официальный образ выполняет envsubst из /etc/nginx/templates/ автоматически

# ── Дополнительные production настройки ─────────────────────────────────────

# /etc/docker/daemon.json — глобальные настройки
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,       # контейнеры живут при рестарте daemon
  "userland-proxy": false,    # использовать hairpin NAT вместо userland proxy
  "no-new-privileges": true,  # глобальный no-new-privs
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
```

> ⚠️ **Антипаттерн:** `CMD ["sh", "-c", "node server.js"]`. Shell-форма CMD/ENTRYPOINT делает `sh` процессом PID 1. SIGTERM приходит на `sh`, который по умолчанию не форвардит его дочернему `node`. Node не получает SIGTERM → Docker ждёт `stop-timeout` → посылает SIGKILL. Всегда используйте exec-форму: `CMD ["node", "server.js"]`.

> ✅ **Senior-совет:** `--live-restore true` в daemon.json позволяет контейнерам продолжать работать при рестарте Docker daemon (обновления, патчи). Критично для production-хостов, где `systemctl restart docker` иначе убивает все контейнеры. Совместимо с большинством конфигураций, но несовместимо с Docker Swarm.

---

## Быстрый справочник

| # | Команда / Флаг | Сценарий |
|---|----------------|----------|
| 1 | `docker run --rm -it` | Одноразовый интерактивный контейнер |
| 2 | `docker run -d --restart unless-stopped` | Фоновый сервис с авторестартом |
| 3 | `docker run --memory 512m --memory-swap 512m` | Лимит памяти без swap |
| 4 | `docker run --cpus 1.5` | Ограничение CPU (1.5 ядра) |
| 5 | `docker run --read-only --tmpfs /tmp` | Read-only FS с tmpfs для /tmp |
| 6 | `docker run --cap-drop ALL --cap-add NET_BIND_SERVICE` | Минимальные capabilities |
| 7 | `docker run --security-opt no-new-privileges:true` | Блок setuid/setgid |
| 8 | `docker run -p 127.0.0.1:3000:3000` | Bind только на localhost |
| 9 | `docker run --init` | Встроенный tini init-процесс |
| 10 | `docker stop --time 30 <cid>` | Graceful stop с таймаутом |
| 11 | `docker logs --tail 100 -f --timestamps` | Live логи с timestamp |
| 12 | `docker stats --no-stream` | Одноразовый снимок ресурсов |
| 13 | `docker exec -it <cid> sh` | Shell без bash |
| 14 | `docker cp <cid>:/path ./local` | Копирование файлов без exec |
| 15 | `docker diff <cid>` | Изменения в writable слое |
| 16 | `docker system df -v` | Дисковое использование Docker |
| 17 | `docker image prune --filter "until=240h"` | Очистка старых образов |
| 18 | `docker buildx build --platform linux/amd64,linux/arm64` | Multi-arch сборка |
| 19 | `docker network create --internal` | Сеть без внешнего доступа |
| 20 | `docker volume create --opt type=nfs` | Volume с NFS backend |

---

## Дополнительные материалы

### Официальная документация

- [Docker Engine overview](https://docs.docker.com/engine/)
- [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose specification](https://docs.docker.com/compose/compose-file/)
- [Docker BuildKit / Buildx](https://docs.docker.com/build/buildkit/)
- [Docker security](https://docs.docker.com/engine/security/)
- [Docker networking](https://docs.docker.com/network/)
- [Docker storage](https://docs.docker.com/storage/)
- [OCI Image Spec](https://github.com/opencontainers/image-spec/blob/main/spec.md)

### Инструменты

- [Hadolint — Dockerfile linter](https://github.com/hadolint/hadolint)
- [Trivy — vulnerability scanner](https://github.com/aquasecurity/trivy)
- [Syft — SBOM generator](https://github.com/anchore/syft)
- [Cosign — container signing](https://github.com/sigstore/cosign)
- [Dive — layer analyzer](https://github.com/wagoodman/dive)
- [Netshoot — network debug](https://github.com/nicolaka/netshoot)
- [Tini — minimal init](https://github.com/krallin/tini)
- [Docker Slim](https://github.com/slimtoolkit/slim)

### Спецификации и стандарты

- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [NIST Container Security Guide (SP 800-190)](https://csrc.nist.gov/publications/detail/sp/800/190/final)
- [seccomp default profile](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json)
- [Linux capabilities man page](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [OCI Runtime Spec](https://github.com/opencontainers/runtime-spec)

---

*Версия: 2.0.0 | Актуально для Docker Engine 25+, Compose V2, BuildKit 0.12+*
