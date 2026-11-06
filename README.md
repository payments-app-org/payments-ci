# payments-ci

Общие шаблоны CI (reusable workflows GitHub Actions) для репозиториев `payments-app-org`. Шаги сборки описаны
один раз здесь, репозитории подключают их одной строкой. Версия шаблонов — тег `v1` (сдвигается на совместимые
изменения); несовместимые — `v2`, и репозитории переходят на него сами.

| Шаблон | Для чего | Что делает |
|---|---|---|
| `java-service.yml` | сервисы | `mvn verify` (unit + Testcontainers) → Trivy по зависимостям → образ → Trivy по образу → в `main` публикация `ghcr.io/payments-app-org/<репозиторий>:<sha>` и `:main` |
| `java-library.yml` | `payments-platform` | `mvn verify`, Trivy; по тегу `vX.Y.Z` — публикация версии в GitHub Packages |
| `frontend.yml` | `payments-frontend` | `npm ci`, `npm run build` (проверка типов и сборка), Trivy |

У каждого шаблона итоговая задача `ok` — её требует защита `main` (проверка «ci / ok»).

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
