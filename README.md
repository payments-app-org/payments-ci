# payments-ci

Общие шаблоны CI (reusable workflows GitHub Actions) для репозиториев `payments-app-org`. Шаги сборки описаны
один раз здесь, репозитории подключают их одной строкой. Версия шаблонов — тег `v1` (сдвигается на совместимые
изменения); несовместимые — `v2`, и репозитории переходят на него сами.

| Шаблон | Для чего | Что делает |
|---|---|---|
| `java-service.yml` | сервисы | `mvn verify` (unit + Testcontainers) → Trivy по зависимостям → образ → Trivy по образу → в `main` публикация `ghcr.io/payments-app-org/<репозиторий>:<sha>` и `:main` и выкатка (`deploy.yml`) |
| `java-library.yml` | `payments-platform` | `mvn verify`, Trivy; по тегу `vX.Y.Z` — публикация версии в GitHub Packages |
| `frontend.yml` | `payments-frontend` | `npm ci`, `npm run build` (проверка типов и сборка), Trivy → образ (nginx со статикой) → Trivy по образу → в `main` публикация и выкатка |
| `deploy.yml` | вызывают `java-service.yml` и `frontend.yml` | после публикации образа в `main` ждёт кнопки (окружение `prod`), затем записывает тег образа в [payments-deploy](https://github.com/payments-app-org/payments-deploy) (`scripts/set-image.sh` → `envs/prod/services/<сервис>.yaml`); дальше Argo CD в кластере выкатывает сервис |

У каждого шаблона итоговая задача `ok` — её требует защита `main` (проверка «ci / ok»). Выкатку `ok` не ждёт: в PR её нет.

**Выкатка — по кнопке.** В репозитории сервиса окружение `prod` с обязательным подтверждением (Settings →
Environments → prod → Required reviewers). Задача `deploy` ждёт его: в запуске CI в `main` — **Review deployments →
prod → Approve and deploy** (аналог `when: manual` в GitLab). История выкаток — раздел Deployments репозитория.
Без окружения с подтверждением GitHub создаст его сам при первом запуске — и выкатка пойдёт без кнопки.

Выкатка пишет в `payments-deploy` ключом развёртывания (deploy key) — его закрытая часть в секрете `DEPLOY_KEY`
каждого репозитория сервиса, шаблоны получают его через `secrets: inherit`. Новый сервис выкатывается, когда он описан
в `payments-deploy` (строка в `envs/prod/apps/services.yaml`, `services/<сервис>.yaml`, `envs/prod/services/<сервис>.yaml`).

Подключение в репозитории сервиса (`.github/workflows/ci.yml`):

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
jobs:
  ci:
    uses: payments-app-org/payments-ci/.github/workflows/java-service.yml@v1
    secrets: inherit
    permissions:
      contents: read
      packages: write
```

Проверить шаблоны локально: `docker run --rm -v "$PWD":/repo -w /repo rhysd/actionlint:1.7.7`.
