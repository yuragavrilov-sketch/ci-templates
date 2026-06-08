# ci-templates

GitLab CI/CD Components для PAY_ALL. Единый источник пайплайнов: сервис подключает компонент
одной строкой, всё остальное (стадии, гейты, деплой) — внутри компонента.

Конвейер: `validate → test → package(Kaniko) → scan(Trivy) → deploy:test(manual) → smoke`.
Принцип build-once / deploy-same-artifact, иммутабельный тег `$CI_COMMIT_SHORT_SHA`.

> Дизайн и решения: `PAY_ALL/docs/superpowers/specs/2026-06-03-gitlab-ci-reference-pipeline-design.md`
> План раскатки: `PAY_ALL/docs/superpowers/plans/2026-06-03-gitlab-ci-reference-java-path.md`

## Компонент `java-spring`

Для бэкендов на Maven (Spring Boot 4 / Java 21).

```yaml
include:
  - component: $CI_SERVER_FQDN/tkbpay/ci-templates/java-spring@1.1.5
    inputs:
      release_name: my-service      # DNS-1123, без подчёркиваний
      with_cert: true               # опц., корп. CA в truststore
```

### Inputs

| Input | По умолчанию | Назначение |
|-------|--------------|------------|
| `release_name` | — (обязателен) | имя Helm-релиза (DNS-1123) |
| `with_cert` | `false` | подмешивать корпоративный CA |
| `values_path` | `helm/values-test.yaml` | путь к values |
| `maven_image` | `maven:3.9-eclipse-temurin-21` | образ сборки |
| `chart_repo` | Harbor chartrepo | репозиторий чарта |
| `chart_name` | `kubernetes/java` | имя чарта |
| `chart_version` | `0.4.0` | версия чарта |
| `namespace` | `test` | k8s namespace |
| `image_registry` | `harbor.online.tkbbank.ru` | Docker registry host (Harbor) |
| `image_registry_project` | `tkbpay` | проект реестра под образы |

`IMAGE` = `<image_registry>/<image_registry_project>/$CI_PROJECT_NAME`. Push в Harbor требует
креды в CI/CD-переменной группы **`HARBOR_DOCKER_CONFIG`** (docker `config.json`) — `package`
пишет их в `/kaniko/.docker/config.json`; в код секрет не выносится.

## Компонент `node-spa`

Для SPA-фронта на Vite (сборка → nginx-образ). Требует опубликованного web-чарта `kubernetes/web`.

```yaml
include:
  - component: $CI_SERVER_FQDN/tkbpay/ci-templates/node-spa@1.1.0
    inputs:
      release_name: payadmin-front
      vite_kc_url: "https://keycloak.test..."
```

### Inputs (node-spa)

| Input | По умолчанию | Назначение |
|-------|--------------|------------|
| `release_name` | — (обязателен) | имя Helm-релиза (DNS-1123) |
| `values_path` | `helm/values-test.yaml` | путь к values |
| `node_image` | `node:22-alpine` | образ сборки |
| `chart_name` | `kubernetes/web` | web/nginx-чарт (не `kubernetes/java`) |
| `chart_version` | `0.1.0` | версия web-чарта |
| `namespace` | `test` | k8s namespace |
| `vite_auth_enabled` / `vite_kc_url` / `vite_kc_realm` / `vite_kc_client_id` | см. spec | build-args для Vite |

Стадии: typecheck → vitest+audit → package(Kaniko + VITE build-args) → Trivy →
deploy:test(manual) → smoke (`GET /`, порт 80).

## Перевыкатка / смена конфигурации без пересборки

- **Откат / повторный деплой:** перезапустить ручную джобу `deploy_test` в нужном (в т.ч.
  прошлом) пайплайне — `helm upgrade` тем же образом, без сборки.
- **Смена только `helm/**`:** коммит, не трогающий `Dockerfile`/`src/**`/`pom.xml`, не запускает
  `package` (правило `rules:changes`).
- **Выкатить произвольный тег + новые values:** «Run pipeline» с переменной
  `DEPLOY_IMAGE_TAG=<sha>` → выполнится только `deploy_only` (`helm upgrade`), сборки не будет.

## Требования к runner'ам и переменным (CI/CD variables группы)

- `HARBOR_DOCKER_CONFIG` — docker `config.json`/robot-creds Harbor для Kaniko-push.
  (Хост/проект реестра — это inputs компонента `image_registry`/`image_registry_project`.)
- `K8S_CONFIG_test` — kubeconfig (file, protected).
- Runner'ы должны уметь запускать Kaniko (`gcr.io/kaniko-project/executor:debug`). Если только
  DinD — заменить job `package` (см. spec, вариант DinD).
- Шаблоны `Jobs/SAST` / `Jobs/Dependency-Scanning` должны быть доступны в инстансе (иначе убрать
  из `include` или пометить `allow_failure`).
