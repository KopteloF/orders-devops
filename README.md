# Orders DevOps

Учебный production-подобный контур вокруг Django REST API заказов. Приложение нужно как нагрузка. Фокус репозитория: Docker, CI/CD, Kubernetes, Helm, мониторинг, Vault, IaC.

[![CI](https://github.com/KopteloF/orders-devops/actions/workflows/ci.yml/badge.svg)](https://github.com/KopteloF/orders-devops/actions/workflows/ci.yml)

Репозиторий: https://github.com/KopteloF/orders-devops

## Приложение

Django REST Framework: прайсы поставщиков (YAML), корзина, заказ, токен-аутентификация, PostgreSQL, Redis.

## Стек

| Слой | Технологии |
|------|------------|
| Контейнеры | Docker, docker-compose |
| Оркестрация | k3s (self-hosted), Yandex Managed Kubernetes, Helm |
| CI/CD | GitHub Actions, зеркало GitLab CI, self-hosted runner для деплоя |
| Registry | GHCR, тег образа = короткий git SHA |
| IaC | Terraform (Yandex Cloud), Ansible |
| Мониторинг | Prometheus, Grafana, Alertmanager (Telegram), node-exporter |
| Секреты | HashiCorp Vault (Raft), Kubernetes auth, Agent Injector |

## CI/CD

Цепочка в `.github/workflows/ci.yml`:

1. lint (flake8)
2. test (pytest + Postgres)
3. build и push образа в GHCR
4. на `master`: deploy с self-hosted runner через `helm upgrade --install`, ожидание `rollout status`

Deploy запускается только при переменной репозитория `DEPLOY_ENABLED=true` (Settings → Secrets and variables → Actions → Variables). Стенд поднимается по необходимости, поэтому без кластера CI остаётся зелёным, а не висит в очереди.

Runner регистрируется на control-ноде с доступом к k3s: `deploy/register_runner.yml` (токен из Settings → Actions → Runners → New self-hosted runner).

Зеркало lint/test/build: `.gitlab-ci.yml`.

## Kubernetes и Helm

Чарт `helm/orders`: app, PostgreSQL, Redis, Service, Ingress, ConfigMap.

В чарте:

- probes (readiness HTTP, liveness TCP)
- requests/limits
- initContainer для `migrate`
- topologySpreadConstraints при нескольких репликах
- опционально Vault Agent Injector через values `vault.enabled`
- Ingress; TLS через cert-manager и ClusterIssuer Let's Encrypt (Traefik на k3s)

Обновление: новый тег образа из CI, `helm upgrade`. Откат: `helm rollback` или предыдущий тег.

Кластеры, на которых гонял:

- self-hosted k3s (локально и на VM в Yandex Cloud)
- Yandex Managed Kubernetes (LoadBalancer снаружи)

Service mesh не использовал. Своих операторов и CRD не писал. Из экосистемы: cert-manager (ClusterIssuer), Vault Injector.

## Секреты

Вариант без Vault: k8s Secret + `envFrom`.  
Вариант с Vault: sidecar/injector, роль по ServiceAccount, секреты в `/vault/secrets` в рантайме.

В облаке отрабатывал auto-unseal Vault через Yandex KMS (sidecar + IAM из metadata VM).

## IaC и облако

- `terraform/` + `deploy_cloud.sh`: VM в YC, Ansible, docker-compose контур
- `deploy_cloud_k8s.sh`: VM + k3s + Helm
- `terraform-mks/` + `deploy_mks.sh`: Managed Kubernetes

Remote state Terraform: Object Storage. Секреты и state в git не кладу.

## Быстрый старт

### Локально

```bash
cp .env.example .env
docker compose up -d --build
docker compose run --rm app python manage.py migrate
# http://localhost:8000/shops
```

### Helm в уже существующий кластер

```bash
helm upgrade --install orders helm/orders -n orders --create-namespace
kubectl -n orders get pods
```

### Облако (нужны ключи YC / S3 для state)

См. скрипты `deploy_cloud.sh`, `deploy_cloud_k8s.sh`, `deploy_mks.sh`. После проверки: `terraform destroy` в соответствующем каталоге.

## Структура

```text
orders/                 Django-приложение
Dockerfile
docker-compose.yml
.github/workflows/ci.yml
.gitlab-ci.yml
helm/orders/            Helm-чарт
deploy/                 Ansible и связанные манифесты
monitoring/             Prometheus, Grafana, Alertmanager
vault/                  Vault prod-манифесты и хелперы
terraform/              IaC под VM + remote state
terraform-mks/          IaC под Managed Kubernetes
backup/                 CronJob бэкапов (учебные)
```

## Осознанные учебные компромиссы

- Vault на одной ноде (не HA)
- часть bootstrap-секретов на этапе обучения упрощена
- sslip.io для HTTPS без своего домена (лимиты Let's Encrypt)

## Лицензия / происхождение приложения

Бизнес-логика orders опирается на учебный дипломный каркас. DevOps-обвязка (CI, Helm, k8s, мониторинг, Vault, облако) собрана отдельно как портфолио-контур.
