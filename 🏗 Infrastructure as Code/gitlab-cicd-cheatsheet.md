---
title: "GitLab CI/CD Pipelines — Senior Engineer Cheatsheet"
version: "2.0"
last_updated: "2025-03-01"
author: "Senior DevOps Engineer"
gitlab_version: "16+"
---

# GitLab CI/CD Pipelines — Исчерпывающая шпаргалка

> Актуально для GitLab 16+. Примеры готовы к production-использованию.
> 💎 — функции GitLab Premium/Ultimate.

---

## Содержание

1. [Архитектура GitLab CI/CD](#1-архитектура-gitlab-cicd)
2. [Структура .gitlab-ci.yml](#2-структура-gitlab-ciyml)
3. [Stages и Jobs](#3-stages-и-jobs)
4. [Переменные и окружения](#4-переменные-и-окружения)
5. [Runners](#5-runners)
6. [Artifacts и Cache](#6-artifacts-и-cache)
7. [Шаблоны и переиспользование](#7-шаблоны-и-переиспользование)
8. [Триггеры и условия запуска](#8-триггеры-и-условия-запуска)
9. [Environments и Deployments](#9-environments-и-deployments)
10. [Безопасность и соответствие](#10-безопасность-и-соответствие)
11. [Оптимизация производительности](#11-оптимизация-производительности)
12. [Мониторинг и отладка](#12-мониторинг-и-отладка)
13. [Предопределённые переменные](#предопределённые-переменные)
14. [Быстрый справочник](#быстрый-справочник)
15. [Дополнительные материалы](#дополнительные-материалы)

---

## 1. Архитектура GitLab CI/CD

Pipeline — это граф выполнения jobs, организованных в stages. GitLab Server оркестрирует выполнение, Runner — агент, который физически выполняет job на целевой инфраструктуре. Взаимодействие Runner ↔ Server идёт через polling по HTTPS (Runner инициирует соединение, не наоборот).

**Жизненный цикл job:** `created → pending → running → (passed | failed | canceled)`

**Типы executor Runner:**

| Executor | Изоляция | Применение |
|---|---|---|
| `shell` | Нет | Legacy, простые скрипты |
| `docker` | Container | Сборка образов, тесты |
| `kubernetes` | Pod | Cloud-native, autoscaling |
| `docker+machine` | VM+Container | Autoscaling на AWS/GCP |

```yaml
# Пример: GitLab Agent для Kubernetes (pull-based деплой)
# .gitlab/agents/production/config.yaml
ci_access:
  projects:
    - id: mygroup/myproject
      environments:
        - name: production
        - name: staging
```

```toml
# config.toml Runner — пример для Kubernetes executor
[[runners]]
  name = "k8s-runner"
  url = "https://gitlab.example.com"
  token = "__RUNNER_TOKEN__"
  executor = "kubernetes"
  [runners.kubernetes]
    namespace = "gitlab-runners"
    image = "alpine:3.19"
    pull_policy = ["always", "if-not-present"]
    [runners.kubernetes.node_selector]
      "node-role" = "ci"
    [runners.kubernetes.resource_limits]
      cpu = "2"
      memory = "4Gi"
    [runners.kubernetes.resource_requests]
      cpu = "500m"
      memory = "512Mi"
```

> **⚠️ Антипаттерн:** Использование shell executor в production без контейнерной изоляции — jobs накапливают артефакты, зависимости конфликтуют между проектами, секреты утекают между запусками. Shell executor допустим только для специализированных задач (например, физические машины, iOS-сборка).

> **✅ Senior-совет:** Настройте `clone_url` в config.toml для Runner, если он находится во внутренней сети и GitLab доступен по другому адресу, чем публичный. Это устраняет проблему DNS-резолюции при git clone внутри runner-пода без изменения DNS инфраструктуры.

---

## 2. Структура .gitlab-ci.yml

`.gitlab-ci.yml` — декларативный конфиг в корне репозитория (или по кастомному пути через Settings → CI/CD). Валидация через **CI Lint** (`/ci/lint`) обязательна перед мержем изменений конфига. GitLab поддерживает наследование конфигов через `include`.

```yaml
# .gitlab-ci.yml — production-ready базовый скелет

# Версионирование: храните в комментарии для отслеживания изменений
# schema_version: 1.4.2

default:
  image: alpine:3.19
  tags:
    - docker
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  interruptible: true
  timeout: 30 minutes

workflow:
  name: '$CI_PIPELINE_SOURCE — $CI_COMMIT_BRANCH'
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
      variables:
        PIPELINE_TYPE: scheduled
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      variables:
        PIPELINE_TYPE: mr
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      variables:
        PIPELINE_TYPE: main
    - if: '$CI_COMMIT_BRANCH'
      when: never  # Запрет branch-пайплайнов при наличии MR

stages:
  - validate
  - build
  - test
  - security
  - package
  - deploy
  - verify

variables:
  DOCKER_BUILDKIT: "1"
  REGISTRY: registry.gitlab.example.com
  APP_NAME: myapp
```

> **⚠️ Антипаттерн:** Хранить токены и пароли напрямую в `.gitlab-ci.yml` или в переменных без маскировки. Это приводит к утечке секретов в логах и в git-истории. Всегда используйте masked + protected variables через GitLab UI или Vault-интеграцию.

> **✅ Senior-совет:** Используйте `workflow: name` с переменными для информативных имён пайплайнов в UI. `$CI_PIPELINE_SOURCE — $CI_COMMIT_BRANCH` мгновенно показывает контекст запуска без захода в детали.

---

## 3. Stages и Jobs

Stages выполняются последовательно; jobs внутри одного stage — параллельно. DAG (Directed Acyclic Graph) через `needs` позволяет запускать job сразу после завершения конкретной dependency, игнорируя порядок stages.

```yaml
stages:
  - build
  - test
  - deploy

# Параллельная матричная сборка
build:backend:
  stage: build
  image: golang:1.22-alpine
  script:
    - go build -ldflags="-s -w -X main.version=$CI_COMMIT_SHORT_SHA" -o bin/app ./cmd/app
  artifacts:
    paths:
      - bin/app
    expire_in: 1 hour

build:frontend:
  stage: build
  image: node:20-alpine
  script:
    - npm ci --cache .npm
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# DAG: test:integration не ждёт всех build jobs — стартует как только нужные готовы
test:unit:backend:
  stage: test
  needs:
    - job: build:backend
      artifacts: true
  image: golang:1.22-alpine
  script:
    - go test ./... -race -coverprofile=coverage.out
    - go tool cover -func=coverage.out | tail -1
  coverage: '/^total:\s+\(statements\)\s+(\d+\.\d+)%/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

test:e2e:
  stage: test
  needs:
    - job: build:backend
      artifacts: true
    - job: build:frontend
      artifacts: true
  image: mcr.microsoft.com/playwright:v1.43.0-jammy
  script:
    - npx playwright test --reporter=html
  artifacts:
    when: always
    paths:
      - playwright-report/
    expire_in: 7 days

# Параллельное выполнение через parallel:matrix
test:cross-platform:
  stage: test
  needs: []
  parallel:
    matrix:
      - OS: [ubuntu, alpine]
        ARCH: [amd64, arm64]
  tags:
    - $ARCH
  script:
    - echo "Testing on $OS/$ARCH"
    - ./scripts/run-tests.sh

deploy:production:
  stage: deploy
  needs:
    - job: test:unit:backend
    - job: test:e2e
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
  environment:
    name: production
```

> **⚠️ Антипаттерн:** Добавлять `needs: []` ко всем jobs, думая что это ускорит пайплайн — при этом теряются гарантии порядка выполнения. Jobs с `needs: []` запускаются немедленно, минуя этапные проверки, что приводит к деплою непроверенного кода.

> **✅ Senior-совет:** `needs` с `artifacts: false` позволяет выстроить зависимость по порядку выполнения без скачивания артефактов. Полезно для job типа "notify", которая должна запускаться после деплоя, но не нуждается в бинарниках.

---

## 4. Переменные и окружения

GitLab CI предоставляет богатый набор предопределённых переменных, group/project-переменных с поддержкой маскировки и защиты. Передача данных между jobs осуществляется через artifacts (файлы) или `dotenv` report (переменные окружения).

```yaml
# Передача динамических переменных между jobs через dotenv
prepare:version:
  stage: validate
  script:
    - VERSION=$(git describe --tags --always --dirty)
    - BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
    - |
      cat > build.env << EOF
      APP_VERSION=${VERSION}
      BUILD_DATE=${BUILD_DATE}
      IMAGE_TAG=${CI_COMMIT_SHORT_SHA}
      EOF
  artifacts:
    reports:
      dotenv: build.env

build:image:
  stage: build
  needs:
    - job: prepare:version
      artifacts: true
  script:
    # APP_VERSION, BUILD_DATE, IMAGE_TAG доступны как переменные окружения
    - echo "Building version $APP_VERSION"
    - |
      docker build \
        --build-arg VERSION="$APP_VERSION" \
        --build-arg BUILD_DATE="$BUILD_DATE" \
        --label "org.opencontainers.image.version=$APP_VERSION" \
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHA" \
        -t "$REGISTRY/$APP_NAME:$IMAGE_TAG" \
        -t "$REGISTRY/$APP_NAME:latest" \
        .
    - docker push "$REGISTRY/$APP_NAME:$IMAGE_TAG"
    - docker push "$REGISTRY/$APP_NAME:latest"
```

```yaml
# Иерархия переменных (приоритет от низкого к высокому):
# 1. Predefined variables (GitLab)
# 2. Group variables
# 3. Project variables
# 4. .gitlab-ci.yml variables:
# 5. Job variables

variables:
  # Глобальные — доступны во всех jobs
  KUBECONFIG_PATH: /tmp/kubeconfig
  HELM_VERSION: "3.14.0"

deploy:staging:
  variables:
    # Job-level переменная перекрывает глобальную
    K8S_NAMESPACE: staging
    REPLICAS: "2"
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - kubectl config set-cluster ... --kubeconfig=$KUBECONFIG_PATH
    - helm upgrade --install $APP_NAME ./chart
        --namespace $K8S_NAMESPACE
        --set image.tag=$CI_COMMIT_SHORT_SHA
        --set replicaCount=$REPLICAS
        --wait --timeout=5m
```

> **⚠️ Антипаттерн:** Использование `variables` с `expand: true` (по умолчанию) для значений, содержащих `$` (например, regex, shell-скрипты, пароли с символами). GitLab интерполирует `$VAR` внутри значений переменных, что ломает regex и приводит к непредсказуемому поведению. Используйте `$` как `$$` или single-quoted shell-строки.

> **✅ Senior-совет:** Для передачи секретов из HashiCorp Vault используйте нативную интеграцию GitLab (💎) или `vault` CLI в скрипте с `CI_JOB_JWT_V2` для аутентификации. Никогда не храните production-секреты в GitLab Variables для критических окружений — используйте External Secret Operator в Kubernetes.

---

## 5. Runners

Runner — это агент, который выполняет jobs. Shared runners предоставляются GitLab.com; specific runners — self-hosted для проекта/группы. Теги позволяют маршрутизировать jobs на нужные runners.

```toml
# config.toml — продакшн конфигурация Runner
concurrent = 10  # Максимум параллельных jobs на этом Runner
check_interval = 3
shutdown_timeout = 30

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-runner-prod"
  url = "https://gitlab.example.com"
  token = "__RUNNER_TOKEN__"
  executor = "docker"
  limit = 5  # Максимум jobs для этого runner

  [runners.docker]
    image = "alpine:3.19"
    privileged = false  # НИКОГДА true в production без необходимости
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock:ro"]
    # Для Docker-in-Docker используйте services, не privileged!
    shm_size = 268435456  # 256MB для тестов с большим shared memory
    pull_policy = ["always"]
    memory = "4g"
    memory_swap = "4g"
    cpus = "2"

  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "eu-west-1"
      AuthenticationType = "iam"  # IAM Role — без хардкода ключей
```

```yaml
# Использование тегов для маршрутизации на специализированные runners
build:ios:
  tags:
    - macos
    - xcode-15
  script:
    - xcodebuild -scheme MyApp -configuration Release

deploy:gpu:
  tags:
    - gpu
    - cuda-12
  script:
    - python train.py --device cuda

# Protected runner: только для деплоя в production
deploy:prod:
  tags:
    - production-deploy  # Runner с этим тегом — protected, только main branch
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

> **⚠️ Антипаттерн:** Запускать `privileged = true` для всех Docker runners ради Docker-in-Docker. Это эквивалентно root-доступу на хосте. Вместо этого используйте `kaniko` или `buildah` для сборки образов без привилегий, или выделите отдельный изолированный runner для DinD с network policy.

> **✅ Senior-совет:** Настройте `FF_USE_FASTZIP=true` и `ARTIFACT_COMPRESSION_LEVEL=fast` в переменных Runner для ускорения загрузки/выгрузки артефактов на 30-60%. Это feature flag GitLab Runner, активируется через переменные окружения в config.toml или в `.gitlab-ci.yml`.

---

## 6. Artifacts и Cache

**Cache** — переиспользование данных между pipeline runs (node_modules, go modules, pip packages). **Artifacts** — передача данных между jobs внутри pipeline и хранение результатов (бинарники, отчёты).

```yaml
# Продвинутая стратегия кэширования
variables:
  CACHE_VERSION: v2  # Инвалидируйте кэш, меняя версию

.go-cache: &go-cache
  cache:
    - key:
        files:
          - go.sum
        prefix: go-modules-$CACHE_VERSION
      paths:
        - .go/pkg/mod/
      policy: pull  # Jobs только читают кэш, не пишут

build:go:
  <<: *go-cache
  cache:
    - key:
        files:
          - go.sum
        prefix: go-modules-$CACHE_VERSION
      paths:
        - .go/pkg/mod/
      policy: pull-push  # Build job обновляет кэш
  variables:
    GOPATH: "$CI_PROJECT_DIR/.go"
  script:
    - go build ./...

# Кэширование Docker слоёв через registry
build:docker:
  image: docker:26
  services:
    - docker:26-dind
  variables:
    DOCKER_BUILDKIT: "1"
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - |
      docker buildx build \
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:buildcache \
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:buildcache,mode=max \
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA \
        --push \
        .

# Artifacts с типизированными reports
test:pytest:
  script:
    - pip install pytest pytest-cov
    - pytest --junitxml=junit.xml --cov=. --cov-report=xml:coverage.xml
  artifacts:
    when: always  # Сохранять даже при падении тестов
    expire_in: 30 days
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - junit.xml
      - coverage.xml
      - htmlcov/
```

> **⚠️ Антипаттерн:** Кэшировать `node_modules` по ключу только из branch — при большом числе веток кэш никогда не переиспользуется. Используйте `files: [package-lock.json]` как ключ: одинаковый lockfile даёт cache hit между branches с одинаковыми зависимостями.

> **✅ Senior-совет:** Для монорепозиториев используйте fallback-ключи кэша с множественными `key` записями в списке. GitLab проверяет ключи по порядку и использует первый найденный. Это позволяет branch-jobs использовать кэш из `main` при первом запуске:

```yaml
cache:
  - key: "$CI_COMMIT_REF_SLUG-$CI_JOB_NAME"
    paths: [.cache/]
    policy: pull-push
  - key: "main-$CI_JOB_NAME"   # Fallback на кэш main-ветки
    paths: [.cache/]
    policy: pull
```

---

## 7. Шаблоны и переиспользование

GitLab поддерживает несколько механизмов переиспользования конфигурации: YAML anchors (локально в файле), `extends` (наследование job), `include` (подключение внешних файлов), `!reference` (ссылка на части других jobs).

```yaml
# include — подключение шаблонов (local, project, remote, template)
include:
  # Локальный файл в репозитории
  - local: '.gitlab/ci/security.gitlab-ci.yml'

  # Файл из другого проекта (с конкретным ref)
  - project: 'devops/ci-templates'
    ref: 'v2.1.0'  # Всегда pinned ref, не latest!
    file:
      - '/templates/docker-build.yml'
      - '/templates/helm-deploy.yml'

  # GitLab встроенные шаблоны
  - template: 'Security/SAST.gitlab-ci.yml'

  # Remote URL (с осторожностью — supply chain attack вектор)
  - remote: 'https://raw.githubusercontent.com/org/repo/v1.0/template.yml'
```

```yaml
# extends — наследование с перекрытием
.deploy-base:
  image: bitnami/kubectl:1.29
  before_script:
    - echo "$KUBE_CONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  after_script:
    - rm -f /tmp/kubeconfig
  script:
    - kubectl rollout status deployment/$APP_NAME -n $K8S_NAMESPACE --timeout=5m

deploy:staging:
  extends: .deploy-base
  environment:
    name: staging
  variables:
    K8S_NAMESPACE: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy:production:
  extends: .deploy-base
  environment:
    name: production
  variables:
    K8S_NAMESPACE: production
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
```

```yaml
# !reference — точечная ссылка на части других jobs (мощнее anchors)
.setup-docker:
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

.cleanup:
  after_script:
    - docker system prune -f

build:image:
  before_script:
    - !reference [.setup-docker, before_script]
  after_script:
    - !reference [.cleanup, after_script]
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

> **⚠️ Антипаттерн:** Использовать `include: remote:` на внешние URL без pin конкретного коммита/тега. Изменение внешнего файла может сломать ваш pipeline или, хуже, внедрить вредоносный код в CI. Всегда используйте `project:` + `ref:` с semver тегом для шаблонов.

> **✅ Senior-совет:** Компонентная библиотека GitLab CI (💎, GA в 16.0) — это эволюция `include: project`. Компоненты имеют версионирование, документацию и тестирование в самом GitLab. Создайте `gitlab.com/your-group/ci-components` с семантическим версионированием и используйте `include: component: gitlab.com/your-group/ci-components/docker-build@v1.2.3`.

---

## 8. Триггеры и условия запуска

`rules` — современная замена устаревшим `only/except`. Поддерживает сложные условия: `if` (переменные/expressions), `changes` (файловые паттерны), `exists` (наличие файла). `workflow:rules` управляет созданием самого pipeline.

```yaml
# Продвинутые rules для реальных сценариев
.rules:deploy-production: &rules-deploy-production
  rules:
    # Ручной деплой только с main
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH && $CI_PIPELINE_SOURCE != "schedule"'
      when: manual
      allow_failure: false
    - when: never

.rules:skip-on-docs: &rules-skip-on-docs
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - "**/*.md"
        - "docs/**/*"
      when: never
    - when: on_success

test:heavy:
  <<: *rules-skip-on-docs
  script:
    - ./run-integration-tests.sh

# Scheduled pipeline с разными задачами
cleanup:registry:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "cleanup"'
  script:
    - ./scripts/cleanup-old-images.sh

# MR pipeline с условиями по меткам
deploy:review:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      exists:
        - 'helm/**'
      when: on_success
    - when: never
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    on_stop: stop:review
    auto_stop_in: 3 days

stop:review:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  script:
    - helm uninstall $APP_NAME-mr-$CI_MERGE_REQUEST_IID --namespace review

# Multi-project trigger (вызов downstream pipeline)
trigger:deploy-infrastructure:
  trigger:
    project: devops/infrastructure
    branch: main
    strategy: depend  # pipeline ждёт результата downstream
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      changes:
        - infrastructure/**/*
```

```yaml
# workflow:rules — контроль создания pipeline
workflow:
  rules:
    # MR pipeline
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    # Scheduled pipeline
    - if: '$CI_PIPELINE_SOURCE == "schedule"'
    # Main branch pipeline
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    # Tags — для release
    - if: '$CI_COMMIT_TAG'
    # Отмена: не создавать branch pipeline если есть MR
    - if: '$CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS'
      when: never
    - if: '$CI_COMMIT_BRANCH'
```

> **⚠️ Антипаттерн:** Смешивать `only/except` и `rules` в одном job или файле. GitLab явно запрещает это и выдаёт ошибку. При миграции с `only/except` — мигрируйте весь файл целиком, используйте `include` для изоляции старых шаблонов во время переходного периода.

> **✅ Senior-совет:** `rules: changes:` с `compare_to: refs/heads/main` гарантирует корректное сравнение даже в scheduled или manual pipelines, где `CI_MERGE_REQUEST_TARGET_BRANCH_SHA` не определён. Без `compare_to` на scheduled pipeline `changes` всегда считает все файлы изменёнными.

---

## 9. Environments и Deployments

Environment в GitLab — это именованное окружение с историей деплоев, rollback, approval и связью с мониторингом. Protected environments (💎) добавляют approval workflow.

```yaml
# Полный цикл deployment: staging → production с approval
deploy:staging:
  stage: deploy
  image: bitnami/helm:3.14
  environment:
    name: staging
    url: https://staging.example.com
    deployment_tier: staging
  before_script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - |
      helm upgrade --install $APP_NAME ./chart \
        --namespace staging \
        --create-namespace \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set ingress.host=staging.example.com \
        --atomic \
        --timeout 10m \
        --history-max 5
    - |
      kubectl rollout status deployment/$APP_NAME \
        --namespace staging \
        --timeout=5m
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# 💎 Protected environment с required approvals
deploy:production:
  stage: deploy
  image: bitnami/helm:3.14
  environment:
    name: production
    url: https://app.example.com
    deployment_tier: production
  before_script:
    - echo "$KUBE_CONFIG_PROD" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - |
      helm upgrade --install $APP_NAME ./chart \
        --namespace production \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set replicaCount=3 \
        --set resources.requests.cpu=500m \
        --set resources.requests.memory=512Mi \
        --atomic \
        --timeout 15m \
        --history-max 10
    # Smoke test после деплоя
    - sleep 30
    - curl -sf https://app.example.com/health | jq -e '.status == "healthy"'
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# Review Apps для каждого MR
deploy:review:
  stage: deploy
  image: bitnami/helm:3.14
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://mr-$CI_MERGE_REQUEST_IID.review.example.com
    on_stop: stop:review
    auto_stop_in: 1 week
    deployment_tier: development
  script:
    - |
      helm upgrade --install "review-$CI_MERGE_REQUEST_IID" ./chart \
        --namespace review \
        --create-namespace \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set ingress.host="mr-$CI_MERGE_REQUEST_IID.review.example.com" \
        --set replicaCount=1
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

stop:review:
  stage: deploy
  image: bitnami/helm:3.14
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  script:
    - helm uninstall "review-$CI_MERGE_REQUEST_IID" --namespace review --ignore-not-found
  when: manual
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

> **⚠️ Антипаттерн:** Использовать `when: manual` для production-деплоя без `allow_failure: false`. Если manual job не была запущена, downstream jobs всё равно запускаются с `when: on_success`, что может привести к неожиданному поведению. Явно ставьте `allow_failure: false` на критичных manual jobs.

> **✅ Senior-совет:** Используйте `deployment_tier` (production, staging, testing, development, other) — это даёт GitLab понять семантику окружения и правильно группирует их в Environments UI. Также позволяет фильтровать деплои в Deployment Frequency метриках DORA.

---

## 10. Безопасность и соответствие

GitLab Ultimate предоставляет встроенный Security Suite. Для GitLab Free/Premium — используйте open-source инструменты с кастомными job definitions и `artifacts: reports:` для интеграции в Security Dashboard.

```yaml
include:
  - template: 'Security/SAST.gitlab-ci.yml'          # 💎 нативная интеграция
  - template: 'Security/Secret-Detection.gitlab-ci.yml'
  - template: 'Security/Container-Scanning.gitlab-ci.yml'  # 💎
  - template: 'Security/Dependency-Scanning.gitlab-ci.yml' # 💎
  - template: 'Security/DAST.gitlab-ci.yml'           # 💎

# Кастомный SAST для Free-тиров (без 💎)
sast:semgrep:
  stage: security
  image: returntocorp/semgrep:1.70.0
  script:
    - |
      semgrep \
        --config=auto \
        --json \
        --output=semgrep-report.json \
        --metrics=off \
        src/
  artifacts:
    reports:
      sast: semgrep-report.json  # Интеграция с GitLab Security tab
    paths:
      - semgrep-report.json
    expire_in: 30 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

# Container Scanning без 💎
scan:container:trivy:
  stage: security
  image:
    name: aquasec/trivy:0.50.1
    entrypoint: [""]
  variables:
    TRIVY_NO_PROGRESS: "true"
    TRIVY_CACHE_DIR: ".trivy-cache"
    FULL_IMAGE_NAME: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  script:
    # Fail на CRITICAL уязвимости
    - trivy image --exit-code 1 --severity CRITICAL --no-progress $FULL_IMAGE_NAME
    # Генерация SARIF для GitLab Security Dashboard
    - trivy image --format sarif --output trivy-results.sarif $FULL_IMAGE_NAME
    # SBOM в CycloneDX формате
    - trivy image --format cyclonedx --output sbom.json $FULL_IMAGE_NAME
  cache:
    paths:
      - .trivy-cache/
  artifacts:
    reports:
      container_scanning: trivy-results.sarif
    paths:
      - sbom.json
      - trivy-results.sarif
    expire_in: 30 days
  needs:
    - job: build:image

# Dependency Check (OWASP)
scan:dependencies:
  stage: security
  image: owasp/dependency-check:9.0.9
  script:
    - |
      /usr/share/dependency-check/bin/dependency-check.sh \
        --project "$CI_PROJECT_NAME" \
        --scan . \
        --format JSON \
        --format HTML \
        --out reports/ \
        --failOnCVSS 8 \
        --enableRetired
  artifacts:
    paths:
      - reports/
    expire_in: 30 days
  cache:
    key: dependency-check-data
    paths:
      - /usr/share/dependency-check/data/

# Подпись артефактов с cosign (SLSA)
sign:image:
  stage: security
  image: bitnami/cosign:2.2.3
  script:
    - cosign sign --yes
        --key env://COSIGN_PRIVATE_KEY
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - cosign attest --yes
        --key env://COSIGN_PRIVATE_KEY
        --type cyclonedx
        --predicate sbom.json
        $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  needs:
    - job: build:image
    - job: scan:container:trivy
      artifacts: true
```

> **⚠️ Антипаттерн:** Запускать security scans только на main branch. К моменту мержа уязвимость уже в истории. Включайте security jobs в MR pipeline с `allow_failure: true` для non-blocking ознакомления, но добавляйте breaking threshold в scheduled pipeline для baseline audit.

> **✅ Senior-совет:** Используйте `SAST_EXCLUDED_PATHS` и `DS_EXCLUDED_PATHS` переменные для исключения test-директорий из сканирования. Ложные срабатывания на тестовый код с хардкодными "секретами" (fixtures) убивают доверие к security pipeline быстрее всего.

---

## 11. Оптимизация производительности

Ключевые метрики: pipeline duration, job queue time, cache hit rate. Инструменты: `interruptible`, `resource_group`, `parallel:matrix`, слоёвое кэширование Docker.

```yaml
# interruptible — отмена устаревших jobs при новом push
.interruptible-defaults:
  interruptible: true  # Pipeline отменяется при новом commit в ту же ветку

# resource_group — сериализация deploys, избежание race conditions
deploy:production:
  resource_group: production  # Одновременно — только 1 deploy в production
  environment:
    name: production

# Параллелизация тестов через matrix
test:pytest-sharded:
  parallel:
    matrix:
      - SHARD_INDEX: ["0", "1", "2", "3"]
  variables:
    PYTEST_SHARD_TOTAL: "4"
  script:
    - |
      pytest tests/ \
        --shard-id=$SHARD_INDEX \
        --num-shards=$PYTEST_SHARD_TOTAL \
        --junitxml=junit-$SHARD_INDEX.xml
  artifacts:
    reports:
      junit: junit-$SHARD_INDEX.xml

# Минимизация времени: использование slim-образов и кэша пакетов
build:npm:
  image: node:20-alpine  # Alpine вместо full debian (~200MB vs ~900MB)
  variables:
    npm_config_cache: "$CI_PROJECT_DIR/.npm"
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - .npm/
  script:
    - npm ci --prefer-offline  # prefer-offline — использует кэш максимально
    - npm run build

# Docker BuildKit — параллельная сборка слоёв, mounted cache
build:docker-optimized:
  image: docker:26
  services:
    - name: docker:26-dind
      command: ["--tls=false"]
  variables:
    DOCKER_HOST: tcp://docker:2375
    DOCKER_BUILDKIT: "1"
    BUILDKIT_INLINE_CACHE: "1"
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - |
      docker build \
        --cache-from $CI_REGISTRY_IMAGE:buildcache \
        --cache-from $CI_REGISTRY_IMAGE:$CI_COMMIT_BEFORE_SHA \
        --build-arg BUILDKIT_INLINE_CACHE=1 \
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA \
        --push \
        .
      # Обновляем buildcache тег
      docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA $CI_REGISTRY_IMAGE:buildcache
      docker push $CI_REGISTRY_IMAGE:buildcache
```

```yaml
# Conditional jobs — пропуск тяжёлых jobs для документационных изменений
.skip-docs-changes:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        paths:
          - "**/*.md"
          - "docs/**"
          - ".gitlab/**/*.md"
        compare_to: "refs/heads/main"
      when: never
    - when: on_success

test:integration:
  extends: .skip-docs-changes
  timeout: 20 minutes
  script:
    - ./run-integration-tests.sh
```

> **⚠️ Антипаттерн:** Использовать `image: ubuntu:latest` или `image: python:latest` — `latest` не кэшируется эффективно (меняется SHA), увеличивает время pull. Всегда используйте конкретные теги образов и прописывайте их в переменных для централизованного обновления.

> **✅ Senior-совет:** Включите `FF_USE_FASTZIP=1` и `FF_NETWORK_PER_BUILD=1` через `variables` в `config.toml` runner. FastZip использует параллельное сжатие для artifacts/cache. Network-per-build даёт каждому job изолированный Docker network, устраняя конфликты портов и утечки сетевого трафика между concurrent jobs.

---

## 12. Мониторинг и отладка

Pipeline debugging: `CI_DEBUG_TRACE`, API для programmatic анализа, retry-стратегии, интеграция метрик Runner с Prometheus.

```yaml
# Retry с указанием причин
.retry-on-infra-failure:
  retry:
    max: 3
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
      - scheduler_failure
      - api_failure

# allow_failure с exit codes — частичная обработка ошибок
test:optional-lint:
  script:
    - golangci-lint run ./... --timeout=5m
  allow_failure:
    exit_codes:
      - 1  # Lint warnings — не блокируем pipeline
      # Exit 2 = критическая ошибка инструмента — блокируем

# Отладка: включение trace только для конкретной ветки
debug:trace:
  variables:
    CI_DEBUG_TRACE: "true"  # ВНИМАНИЕ: маскированные переменные будут видны в логах!
  script:
    - env | sort  # Вывод всех переменных для отладки
  rules:
    - if: '$CI_COMMIT_BRANCH == "debug-ci"'
      when: always
    - when: never
```

```bash
# Анализ pipeline через GitLab API
GITLAB_URL="https://gitlab.example.com"
PROJECT_ID="123"
TOKEN="$GITLAB_API_TOKEN"

# Последние 10 failed pipelines
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/pipelines?status=failed&per_page=10" \
  | jq '.[].id'

# Самые медленные jobs за последние 30 дней
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/jobs?per_page=100" \
  | jq '[.[] | select(.status=="success") | {name: .name, duration: .duration}]
        | sort_by(-.duration) | .[0:10]'

# Trigger manual pipeline через API (полезно для automation)
curl -s --request POST \
  --header "PRIVATE-TOKEN: $TOKEN" \
  --header "Content-Type: application/json" \
  --data '{
    "ref": "main",
    "variables": [
      {"key": "SCHEDULE_TYPE", "value": "cleanup"},
      {"key": "DRY_RUN", "value": "true"}
    ]
  }' \
  "$GITLAB_URL/api/v4/projects/$PROJECT_ID/pipeline"
```

```yaml
# Интеграция метрик Runner с Prometheus
# Добавьте в config.toml:
# listen_address = ":9252"  # Prometheus metrics endpoint

# Метрики доступны по /metrics:
# gitlab_runner_jobs_total — счётчик jobs
# gitlab_runner_job_duration_seconds — latency
# gitlab_runner_errors_total — ошибки

# Grafana dashboard: import ID 9826 (GitLab Runner)
```

```yaml
# Slack/webhook уведомления о падениях pipeline
notify:failure:
  stage: .post
  image: curlimages/curl:8.6.0
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-type: application/json' \
        -d "{
          \"text\": \"❌ Pipeline failed!\",
          \"attachments\": [{
            \"color\": \"danger\",
            \"fields\": [
              {\"title\": \"Project\", \"value\": \"$CI_PROJECT_NAME\", \"short\": true},
              {\"title\": \"Branch\", \"value\": \"$CI_COMMIT_BRANCH\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"$CI_COMMIT_SHORT_SHA\", \"short\": true},
              {\"title\": \"Author\", \"value\": \"$CI_COMMIT_AUTHOR\", \"short\": true},
              {\"title\": \"Pipeline URL\", \"value\": \"$CI_PIPELINE_URL\"}
            ]
          }]
        }"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: on_failure
```

> **⚠️ Антипаттерн:** Включать `CI_DEBUG_TRACE: "true"` глобально или в переменных проекта — все masked переменные (пароли, токены) будут отображаться в логах открытым текстом. Debug trace — только для отдельного job в защищённой ветке с последующим немедленным удалением.

> **✅ Senior-совет:** Используйте `artifacts: reports: metrics:` для экспорта кастомных метрик в GitLab (📊 в pipeline view). Создайте файл `metrics.txt` в формате Prometheus text format — GitLab парсит его и показывает trend между пайплайнами. Идеально для отслеживания размера бинарника, покрытия тестами, числа ошибок lint.

```yaml
collect:metrics:
  script:
    - echo "binary_size_bytes $(stat -c%s bin/app)" > metrics.txt
    - echo "test_count $(go test ./... 2>&1 | grep -c 'PASS')" >> metrics.txt
  artifacts:
    reports:
      metrics: metrics.txt
```

---

## Предопределённые переменные

Наиболее полезные встроенные переменные GitLab CI для production-использования:

| Переменная | Описание | Сценарий применения |
|---|---|---|
| `$CI_COMMIT_SHA` | Полный SHA коммита (40 символов) | Docker image label, audit trail |
| `$CI_COMMIT_SHORT_SHA` | Короткий SHA (8 символов) | Docker image tag, version string |
| `$CI_COMMIT_BRANCH` | Имя ветки (не в MR pipeline) | Условия rules, имена окружений |
| `$CI_COMMIT_TAG` | Тег (только для tag pipeline) | Release деплой, semver версии |
| `$CI_COMMIT_REF_SLUG` | Branch/tag в URL-safe формате | Kubernetes namespace, DNS-имена |
| `$CI_COMMIT_MESSAGE` | Сообщение коммита | Changelog, changelog generation |
| `$CI_PIPELINE_SOURCE` | Источник: push/schedule/trigger/merge_request_event | rules conditions |
| `$CI_PIPELINE_URL` | URL пайплайна в GitLab | Slack-уведомления, audit logs |
| `$CI_JOB_NAME` | Имя текущего job | Cache key, отладка |
| `$CI_JOB_TOKEN` | Токен job для GitLab API | Clone других репозиториев, Registry auth |
| `$CI_JOB_URL` | URL job | Уведомления о падениях |
| `$CI_PROJECT_ID` | Числовой ID проекта | GitLab API calls |
| `$CI_PROJECT_NAME` | Имя проекта | Docker image name, уведомления |
| `$CI_PROJECT_PATH` | Полный путь `group/project` | Registry URL, clone URL |
| `$CI_PROJECT_URL` | URL проекта в GitLab | Ссылки в уведомлениях |
| `$CI_REGISTRY` | Адрес GitLab Container Registry | `docker login $CI_REGISTRY` |
| `$CI_REGISTRY_IMAGE` | Полный путь к image `registry/group/project` | Docker build/push |
| `$CI_REGISTRY_USER` | Пользователь для Registry | `docker login` |
| `$CI_REGISTRY_PASSWORD` | Пароль для Registry | `docker login` |
| `$CI_ENVIRONMENT_NAME` | Имя окружения | Конфигурация деплоя |
| `$CI_ENVIRONMENT_SLUG` | URL-safe имя окружения | Kubernetes namespace |
| `$CI_MERGE_REQUEST_IID` | Номер MR | Review app naming, комментарии |
| `$CI_MERGE_REQUEST_TARGET_BRANCH_NAME` | Целевая ветка MR | rules: changes: compare_to |
| `$CI_DEFAULT_BRANCH` | Дефолтная ветка репозитория | rules вместо хардкода "main" |
| `$GITLAB_USER_EMAIL` | Email пользователя, запустившего pipeline | Аудит, git config для commits |

---

## Быстрый справочник

| Ключ | Тип | Описание |
|---|---|---|
| `stages` | `list[string]` | Определение порядка стадий pipeline |
| `image` | `string \| object` | Docker-образ для выполнения job |
| `services` | `list[string \| object]` | Дополнительные контейнеры (БД, DinD) |
| `before_script` | `list[string]` | Команды перед `script` (глобально или на job) |
| `script` | `list[string]` | Основные команды job (обязательно) |
| `after_script` | `list[string]` | Команды после `script` (выполняются всегда) |
| `variables` | `map[string]string` | Переменные окружения (глобальные или job-level) |
| `rules` | `list[object]` | Условия запуска job (современный подход) |
| `needs` | `list[string \| object]` | DAG-зависимости: запуск без ожидания stage |
| `dependencies` | `list[string]` | Список jobs, чьи artifacts скачать |
| `artifacts` | `object` | Сохранение файлов/отчётов между jobs |
| `cache` | `object \| list[object]` | Кэширование директорий между pipeline runs |
| `environment` | `string \| object` | Привязка к named environment (деплой) |
| `extends` | `string \| list[string]` | Наследование конфигурации job |
| `include` | `list[object]` | Подключение внешних CI-конфигов |
| `trigger` | `object` | Запуск downstream pipeline |
| `parallel` | `integer \| object` | Параллельный запуск copies или matrix |
| `retry` | `integer \| object` | Автоповтор при падении |
| `timeout` | `string` | Таймаут job (например `30 minutes`) |
| `allow_failure` | `boolean \| object` | Игнорировать падение job |
| `when` | `string` | `on_success \| on_failure \| always \| manual \| delayed \| never` |
| `interruptible` | `boolean` | Отмена при новом pipeline для той же ref |
| `resource_group` | `string` | Сериализация jobs с одним ресурсом |
| `tags` | `list[string]` | Маршрутизация на конкретный Runner |
| `coverage` | `string` | Regex для парсинга % покрытия из лога |
| `workflow` | `object` | Глобальные правила создания pipeline |
| `default` | `object` | Дефолтные значения для всех jobs |
| `!reference` | `tag` | Ссылка на часть конфигурации другого job |

---

## Дополнительные материалы

### Официальная документация

- [GitLab CI/CD Reference](https://docs.gitlab.com/ee/ci/yaml/) — полный справочник `.gitlab-ci.yml`
- [GitLab Runner docs](https://docs.gitlab.com/runner/) — установка, конфигурация, executor
- [CI/CD Variables](https://docs.gitlab.com/ee/ci/variables/) — переменные и predefined variables
- [GitLab CI/CD Components](https://docs.gitlab.com/ee/ci/components/) — компонентная библиотека 💎
- [GitLab Agent for Kubernetes](https://docs.gitlab.com/ee/user/clusters/agent/) — GitOps деплой
- [Security Scanner Integration](https://docs.gitlab.com/ee/development/integrations/secure.html) — интеграция сторонних сканеров

### Шаблоны и примеры

- [GitLab CI Templates](https://gitlab.com/gitlab-org/gitlab/-/tree/master/lib/gitlab/ci/templates) — официальные встроенные шаблоны
- [CI Component Catalog](https://gitlab.com/explore/catalog) — каталог переиспользуемых компонентов
- [Auto DevOps](https://docs.gitlab.com/ee/topics/autodevops/) — полная автоматизация без конфига

### Инструменты

- **CI Lint** — `/ci/lint` в любом проекте GitLab (встроенная валидация)
- [GitLab-ci-local](https://github.com/finitec-inc/gitlab-ci-local) — локальный запуск pipeline без push
- [GitLab API Explorer](https://gitlab.example.com/-/graphql-explorer) — GraphQL API для автоматизации
- [Trivy](https://github.com/aquasecurity/trivy) — Container/IaC/SBOM scanning
- [Cosign](https://github.com/sigstore/cosign) — подпись контейнеров (SLSA)
- [Semgrep](https://semgrep.dev/) — SAST без GitLab Ultimate
- [kaniko](https://github.com/GoogleContainerTools/kaniko) — rootless Docker build в Kubernetes

### Метрики и мониторинг

- [GitLab Runner Prometheus Metrics](https://docs.gitlab.com/runner/monitoring/) — экспортируемые метрики
- [Grafana Dashboard #9826](https://grafana.com/grafana/dashboards/9826) — GitLab Runner dashboard
- [DORA Metrics in GitLab](https://docs.gitlab.com/ee/user/analytics/dora_metrics.html) — Deployment Frequency, Lead Time 💎
