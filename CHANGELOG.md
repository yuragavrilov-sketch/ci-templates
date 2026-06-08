# Changelog

## [Unreleased]

## [1.1.9]

### Changed
- `java-spring`: `container_scan` → `allow_failure: true` (закрытый контур: trivy-db с ghcr
  недоступна, не блокируем деплой). Inputs `trivy_db_repository`/`trivy_java_db_repository` для
  зеркала БД в Harbor. PROD: зеркалить DB и снять allow_failure.

## [1.1.8]

### Fixed
- `java-spring`: `container_scan` подкладывает `HARBOR_DOCKER_CONFIG` (приватный Harbor pull) и
  `trivy --insecure` (внутренний CA, `x509`). Accepted-for-test. PROD: примонтировать корп-CA.

## [1.1.7]

### Fixed
- `java-spring`: `package` при `with_cert: true` кладёт корп-CA из `$CERT` в `certs/` (Dockerfile
  делает `COPY certs/` и импортит keytool'ом) — иначе сборка падает «no corporate CA found».
- `java-spring`: `test` отдаёт артефактом `deploy/*.jar` (boot-jar из spring-boot repackage), а не
  `target/*.jar` — его COPY'ит Dockerfile.

## [1.1.6]

### Fixed
- `java-spring`: `package` (Kaniko) с `--skip-tls-verify`/`--skip-tls-verify-pull` — Harbor на
  внутреннем CA (`x509: signed by unknown authority`). Accepted-for-test, как helm
  `--insecure-skip-tls-verify`. PROD-харденинг: примонтировать корп-CA в `/kaniko/ssl/certs/`.

## [1.1.5]

### Fixed
- `java-spring`: `package` определяет формат `HARBOR_DOCKER_CONFIG` — путь(File) / сырой JSON / 
  base64 (корп-значение base64-кодировано: `base64 -d` перед записью в config.json). Раньше base64
  писался как есть → Kaniko `invalid character 'e'`.

## [1.1.4]

### Added
- `java-spring`: inputs `image_registry` (default `harbor.online.tkbbank.ru`) и
  `image_registry_project` (default `tkbpay`) — `IMAGE` строится из них, а не из CI/CD-переменных
  `IMAGE_REGISTRY`/`IMAGE_REGISTRY_PROJECT`. Переопределяется на сервис через inputs.
- `java-spring`: `package` пишет креды Harbor из CI/CD-переменной `HARBOR_DOCKER_CONFIG` в
  `/kaniko/.docker/config.json` (GitLab сам подкладывает только `DOCKER_AUTH_CONFIG`). Robust к
  типу переменной (File→путь / обычная→JSON).

## [1.1.3]

### Fixed
- `java-spring`: `container_scan` гейтится теми же `rules:changes`, что `package` — scan только
  когда образ реально собран (иначе тянет несуществующий для sha образ → docker/remote ошибки).

### Notes
- Требует group/project CI/CD-переменных `IMAGE_REGISTRY` и `IMAGE_REGISTRY_PROJECT` (Harbor).
  Без них `IMAGE` = `//<service>` → push/scan/deploy образа не работают. Раньше их давал старый
  шаблон `microservices/template`.

## [1.1.2]

### Fixed
- `java-spring`: `container_scan` сбрасывает entrypoint образа trivy (`entrypoint: [""]`) — иначе
  шелл-скрипт джобы уходит в `trivy` как аргументы (`unknown command "sh"`).

## [1.1.1]

### Fixed
- `java-spring`: `MAVEN_OPTS` больше не содержит `-B` (флаг Maven, не JVM — давал
  `Unrecognized option: -B / Could not create the JVM`); `-B` остаётся только в командной строке `mvn`.
- `java-spring`: `vault_policy_validate` использует `vault policy fmt` (без несуществующего флага
  `-check`).
- `java-spring`: `validate` гонит `mvn -B compile` (без `checkstyle:check` — плагина нет в pom'ах).

## [1.1.0]

### Added
- `node-spa` — CI/CD-компонент для SPA-фронта (Vite → nginx): typecheck / vitest+audit /
  package(Kaniko + VITE build-args) / scan(Trivy) / deploy:test(manual, web-чарт) / smoke (GET /).
- Generic web/nginx Helm-чарт публикуется как `kubernetes/web` (см. `charts/web/`).
- `java-spring`: поддержка Vault policy-as-code — джоба `vault_policy_validate` (валидация
  `vault/policy.hcl`, без токена) + `vault_apply` (manual-триггер привилегированного `vault-config`).
  Применение политик — централизованно (least-privilege).

### Migration
- Сервисы, которым нужны vault-джобы, поднимают пин компонента: `java-spring@1.0.0` → `@1.1.0`
  в `.gitlab-ci.yml`. Без бампа версии `vault_policy_validate`/`vault_apply` не появятся, даже если
  в репо есть `vault/policy.hcl`.

## [1.0.0]

### Added
- `java-spring` — первый CI/CD-компонент: стадии validate / test / package(Kaniko) /
  scan(Trivy) / deploy:test(manual) / smoke.
- `_deploy.yml` — общие скрытые job'ы `.helm_deploy` и `.smoke`.
- `rules:changes` на `package` + джоба `deploy_only` — перевыкатка/смена values без пересборки.

### Notes
- SAST / Dependency-Scanning подключаются стандартными шаблонами GitLab (требуют доступности
  в self-managed инстансе).
- Сборка образа — Kaniko (rootless). Для DinD-раннеров заменить job `package`.

<!-- При первом релизе перенести из Unreleased в раздел версии и протегировать `1.0.0`. -->
