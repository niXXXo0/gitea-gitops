# gitea-gitops

Развёртывание gitea (Helm-чарт [gitea-helm](https://github.com/niXXXo0/gitea-helm):
Gitea + PostgreSQL + backup CronJob) в кластере dev (kind) через Flux CD.

## Как работает

- Flux (контроллеры в `flux-system`) следит за этим репозиторием и приводит
  кластер к его состоянию.
- `init-apps.yaml` подключает репо: источник `gitea-gitops` + kustomization
  на `./clusters/dev/gitea`.
- В `./clusters/dev/gitea` лежат: источник чарта (GitHub-репо `gitea-helm`,
  public, ветка `main`), HelmRelease и значения среды.
- HelmRelease ставит чарт из репозитория чарта. Значения собираются из
  `values.yaml` (через ConfigMap) и секрета `gitea-db` (пароль БД).
- Секреты живут в `init-secrets.yaml` под git-crypt и применяются вручную —
  Flux их не видит.

## Развёртывание с нуля

Требуется: kind-кластер, `kubectl`, `flux` CLI, `git-crypt`; в репозитории
должен быть deploy key (read-only), публичная часть — `~/.ssh/gitea-gitops-deploy.pub`.

```bash
# 1. Клонировать репозиторий и расшифровать секреты
git clone git@github.com:niXXXo0/gitea-gitops.git && cd gitea-gitops
git-crypt unlock ~/.config/git-crypt/gitea-gitops.key

# 2. Поднять контроллеры Flux
flux install

# 3. Namespace, секреты, подключение репозитория
kubectl apply -f init-namespace.yaml
kubectl apply -f init-secrets.yaml
kubectl apply -f init-apps.yaml

# 4. Проверка
flux get sources git -n gitea
flux get helmreleases -n gitea
kubectl -n gitea get pods
```

Gitea открывается через `kubectl port-forward -n gitea svc/gitea 3000:3000`
→ http://127.0.0.1:3000

## Эксплуатация

| Действие | Как |
|---|---|
| настройки среды | правишь `clusters/dev/gitea/values.yaml` → commit → push; Flux применит сам |
| изменение чарта | push в `gitea-helm` (main) — Flux подхватит автоматически |
| смена секретов | правишь `init-secrets.yaml` → `kubectl apply -f init-secrets.yaml` |
| откат | `git revert` → push |
| форс-реконсиляция | `flux reconcile kustomization gitea -n gitea` |

⚠️ Ручные `helm upgrade` / `helm rollback` не использовать — Flux перезапишет
состоянием из git.

## Структура

```
init-namespace.yaml    Namespace gitea — вручную, один раз
init-secrets.yaml      секреты (git-crypt) — вручную: пароль БД, deploy key
init-apps.yaml         GitRepository + Flux Kustomization — вручную, один раз
clusters/dev/gitea/    то, что применяет Flux:
                       kustomization, gitrepository, helmrelease, values
```

## Секреты

`init-secrets.yaml` зашифрован git-crypt (правило в `.gitattributes`).
Ключ расшифровки — `~/.config/git-crypt/gitea-gitops.key`: хранить в менеджере
паролей, в git не коммитить. Потеря ключа = потеря доступа к секретам в истории.
