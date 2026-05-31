# Лабораторная работа: Запуск микросервисного приложения в Kubernetes

Выполнил: Крылов Даниил Федорович, М8О-106БВ-25.

## Описание
В этой лабораторной работе разворачивается микросервисное приложение-мессенджер в Kubernetes кластере.

### Используются:
- `Kubernetes` (Docker Desktop) для оркестрации сервисов
- `Kustomize` для разделения конфигов `base`, `dev` и `prod`
- `Argo CD` для GitOps-деплоя
- `S3 CSI` для хранения загружаемых файлов из `message-service`
  
## Цель работы
Подготовить инфраструктурную конфигурацию для запуска приложения в Kubernetes и продемонстрировать:
- развертывание всех сервисов приложения
- применение миграций базы данных
- подключение файлового хранилища через S3 CSI
- настройка `nodeAffinity`
- поддержку окружений `dev` и `prod` через `kustomize`
- GitOps-синхронизацию через Argo CD

## Дополнительная информация

Подробности реализации, особенности настройки CSI-драйвера и дополнительные скриншоты находятся в `docs/report.md`

## Состав приложения

* `frontend` - SPA интерфейс пользователя
* `bff` - Backend For Frontend, единая точка входа frontend
* `user-service` - сервис пользователей
* `message-service` - сервис сообщений и файлов
* `postgres` - база данных
* `migrate-users` - job для миграций БД пользователей
* `migrate-messages` - job для миграций БД сообщений
* `minio` - локальное S3-хранилище файлов для `message-service`

## Структура репозитория

```
.
├── argocd/                  # Application-манифест Argo CD
├── bff/                     # исходный код BFF
├── docs/                    # материалы, пояснения и скриншоты
├── frontend/                # frontend-приложение
├── k8s/
│   ├── base/                # базовые Kubernetes-манифесты
│   └── overlays/
│       ├── dev/             # конфигурация для dev
│       └── prod/            # конфигурация для prod
├── message-service/         # сервис сообщений
├── user-service/            # сервис пользователей
├── docker-compose.yml       # локальный запуск приложения
└── README.md
```

## Используемые образы

Для лабораторной используются готовые образы:

* `mablinov2704/frontend:latest`
* `mablinov2704/bff:latest`
* `mablinov2704/user-service:latest`
* `mablinov2704/message-service:latest`

Дополнительно используются:

* `postgres:16-alpine`
* `ghcr.io/kukymbr/goose-docker:latest`
* `minio/minio:latest`

## Локальный запуск (Docker Desktop)

Предварительно у вас должен быть установлен и включен Kubernetes в настройках Docker Desktop, а также установлен `csi-драйвер`: [ch.ctrox.csi.s3-driver](https://github.com/yyeart/csi-s3).

1. Маркируем единственную ноду кластера `docker-desktop` всеми необходимыми метками для корректной работы `nodeAffinity`:

```
kubectl label nodes docker-desktop workload=system workload=app disk=fast
```

2. Поднимем `minio` и создадим `bucket`:

```
kubectl apply -f k8s/overlays/dev/namespace.yaml
kubectl apply -k k8s/base/minio -n messager
kubectl get pods -n messager -w

kubectl port-forward svc/minio 9001:9001 -n messager
```

Переходим на `localhost:9001`, авторизуемся (логин: `messager-key`, пароль: `messager-secret-key`).
Создаем bucket под названием `messager-uploads`.

3. Поднимем остальные сервисы через Argo CD:
В каталоге `argocd/` находится манифест `application.yaml`. Он настраивает GitOps-деплой из Git-репозитория в Kubernetes.

Установим сам Argo CD:

```
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f [https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml)
```

Прокинем порт и авторизуемся:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Узнаем первоначальный пароль:

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo

```

Переходим на `localhost:8080` и авторизуемся через `admin` и полученный пароль.

Применяем манифест:

```
kubectl apply -f argocd/application.yaml
```

Ожидаемый статус в интерфейсе Argo CD:

* статус `Synced`
* статус `Healthy`

## S3 (MinIO + CSI)

Для `message-service` загрузка файлов настроена через смонтированное S3-хранилище.

Ожидаемое поведение:

* Сервис пишет файлы не на локальный volume, а в каталог, подключенный через S3 CSI (используется `s3fs`).
* После загрузки файл физически появляется в бакете `messager-uploads` в MinIO.
* Путь к каталогу задается через переменную окружения `UPLOADS_DIR`.

## Node Affinity

В лабораторной настроено размещение сервисов по узлам (в рамках Docker Desktop всё запускается на одной ноде, имитируя распределение):

* `postgres` и `minio` требуют узлы с меткой `workload=system`.
* Прикладные сервисы требуют метку `workload=app`.
* Для `message-service` дополнительно задается предпочтение узлов с меткой `disk=fast`.

## Доступ к приложению

Для доступа к frontend-части приложения необходимо пробросить порт:

```
kubectl port-forward svc/frontend 8081:80 -n messager
```

После этого мессенджер будет доступен в браузере по адресу `http://localhost:8081`.