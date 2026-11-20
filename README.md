# gitea-gitops

Flux-репозиторий для развёртывания [gitea-helm](https://github.com/niXXXo0/gitea-helm)
(Gitea + PostgreSQL + backup CronJob) в кластер dev (kind). Паттерн структуры —
как у `webant-core-*` (init-файлы вручную + `clusters/<cluster>/<app>/` под Flux).

## Состав

- **gitea** — чарт `gitea-helm` из GitHub-репозитория niXXXo0 (ветка `main`),
  подключён через `GitRepository` по HTTPS (репозиторий public).
- Бэкап — CronJob внутри чарта (архив файлов + `pg_dump` в PVC, retention 14 дней).

## Структура

```
.gitattributes               # init-secrets.yaml шифруется git-crypt
init-namespace.yaml          # Namespace gitea — применяется вручную, один раз
init-secrets.yaml            # ВСЕ секреты (git-crypt): пароль БД, deploy key
init-apps.yaml               # GitRepository (self) + Kustomization → ./clusters/dev/gitea
clusters/dev/gitea/
├── kustomization.yaml       # ресурсы + configMapGenerator (gitea-values из values.yaml)
├── gitrepository.yaml       # источник чарта gitea-helm
├── helmrelease.yaml         # релиз: values из ConfigMap + секрета gitea-db
└── values.yaml              # НЕ-секретные значения среды
```

## Секреты

Все секреты — в `init-secrets.yaml`, зашифрован **git-crypt** (см. `.gitattributes`).
Flux с этим файлом не работает: человек расшифровывает и применяет вручную,
кластер получает уже открытые Secret'ы.

Работа с секретами:

```bash
# разблокировать репозиторий (после клона):
git-crypt unlock ~/.config/git-crypt/gitea-gitops.key

# поменять секрет: поправить init-secrets.yaml и применить:
kubectl apply -f init-secrets.yaml
```

Ключ `gitea-gitops.key` — симметричный ключ git-crypt, хранить вне репозитория
(менеджер паролей). Потеря = потерять доступ к секретам в истории.

## Установка (после появления remote-репозитория)

1. Создать пустой репозиторий `gitea-gitops` на GitHub, запушить этот репо.
2. Добавить в репозиторий deploy key (read-only) — публичная часть
   `~/.ssh/gitea-gitops-deploy.pub` (Settings → Deploy keys).
3. Поднять контроллеры: `flux install`
4. `kubectl apply -f init-namespace.yaml`
5. Расшифровать и применить секреты:
   `git-crypt unlock ~/.config/git-crypt/gitea-gitops.key && kubectl apply -f init-secrets.yaml`
6. `kubectl apply -f init-apps.yaml`
7. Проверка: `flux get sources git -n gitea`, `flux get helmreleases -n gitea`

## Проверка

```bash
kubectl -n gitea get gitrepositories,kustomizations,helmreleases
kubectl -n gitea get pods,pvc
```

## Обновление

- значения среды (не секретные) — `clusters/dev/gitea/values.yaml`;
- версия чарта — пуш в репозиторий `gitea-helm` (ветка main);
- секреты — `init-secrets.yaml` (вручную, см. выше);
- откат — `git revert` в соответствующем репозитории.
