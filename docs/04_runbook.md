# AI Ops Runbook — onbording-vm01

> Живой диагностический справочник для кластера `onbording-vm01`.  
> Все команды проверены на реальном стеке: Kubernetes 1.29 / Helm / FluxCD / Prometheus / OpenSearch.  
> Запускать в порядке сверху вниз. Каждый сценарий — самодостаточный.

---

## Содержание

1. [Быстрая проверка кластера](#1-быстрая-проверка-кластера)
2. [Pod не стартует / CrashLoopBackOff](#2-pod-не-стартует--crashloopbackoff)
3. [Flux не применяет изменения](#3-flux-не-применяет-изменения)
4. [HelmRelease завис в Progressing или Failed](#4-helmrelease-завис-в-progressing-или-failed)
5. [service-py недоступен](#5-service-py-недоступен)
6. [service-go недоступен](#6-service-go-недоступен)
7. [Kafka / RabbitMQ / Redis / Mongo / Postgres упали](#7-kafka--rabbitmq--redis--mongo--postgres-упали)
8. [OOMKilled — под убит по памяти](#8-oomkilled--под-убит-по-памяти)
9. [Node под давлением (DiskPressure / MemoryPressure)](#9-node-под-давлением-diskpressure--memorypressure)
10. [OpenSearch не принимает логи](#10-opensearch-не-принимает-логи)
11. [Vault не отдаёт секреты](#11-vault-не-отдаёт-секреты)
12. [Prometheus не скрейпит метрики](#12-prometheus-не-скрейпит-метрики)
13. [Полный rollback сервиса](#13-полный-rollback-сервиса)

---

## 1. Быстрая проверка кластера

Первое что делаем при любом инциденте — общая картина за 30 секунд.

```bash
# Состояние ноды
kubectl get nodes -o wide

# Все поды — смотрим не-Running статусы
kubectl get pods -A | grep -v Running | grep -v Completed

# События последних 10 минут по всему кластеру
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# Статус Flux
flux get all -A

# Статус HelmRelease
kubectl get helmrelease -n infra
```

**Ожидаемый здоровый вывод:**
- Нода `onbording-vm01` в статусе `Ready`
- Все поды в `Running`, RESTARTS не растут
- Flux: все ресурсы `True / Applied`
- HelmRelease: `service-py` и `service-go` в `Ready`

---

## 2. Pod не стартует / CrashLoopBackOff

### Симптом
```
NAME          READY   STATUS             RESTARTS
service-py    0/1     CrashLoopBackOff   5
```

### Диагностика

```bash
# Шаг 1 — смотрим Events пода, там причина
kubectl describe pod -l app=service-py -n infra

# Шаг 2 — логи текущего контейнера
kubectl logs -l app=service-py -n infra

# Шаг 3 — логи предыдущего запуска (до крэша)
kubectl logs -l app=service-py -n infra --previous

# Шаг 4 — если OOMKilled смотрим лимиты
kubectl get pod -l app=service-py -n infra -o jsonpath='{.items[0].spec.containers[0].resources}'
```

### Частые причины и фиксы

**ImagePullBackOff** — образ не скачался:
```bash
# Проверяем тег образа в HelmRelease
kubectl get helmrelease service-py -n infra -o yaml | grep image

# Проверяем доступность registry (с ноды)
ssh onbording-vm01 "curl -s https://registry-1.docker.io/v2/"
```

**CrashLoopBackOff — ошибка в приложении:**
```bash
# Смотрим последние строки лога
kubectl logs -l app=service-py -n infra --previous --tail=50
```

**Env не инжектирован (Vault Agent не отработал):**
```bash
# Проверяем init-контейнер vault-agent
kubectl describe pod -l app=service-py -n infra | grep -A5 "Init Containers"
kubectl logs -l app=service-py -n infra -c vault-agent-init
```

---

## 3. Flux не применяет изменения

### Симптом
Запушил коммит в `infra.git` — в кластере ничего не изменилось через 2+ минуты.

### Диагностика

```bash
# Статус source-controller — видит ли репозиторий
flux get source git -n flux-system

# Статус kustomization
flux get kustomization -n flux-system

# Статус HelmRelease
flux get helmrelease -n infra

# Логи source-controller
kubectl logs -n flux-system -l app=source-controller --tail=30

# Логи helm-controller
kubectl logs -n flux-system -l app=helm-controller --tail=30
```

### Форс-reconcile (если Flux завис)

```bash
# Форсируем синхронизацию репозитория
flux reconcile source git flux-system -n flux-system

# Форсируем применение kustomization
flux reconcile kustomization flux-system -n flux-system

# Форсируем конкретный HelmRelease
flux reconcile helmrelease service-py -n infra
flux reconcile helmrelease service-go -n infra
```

### Suspend / Resume (если нужно временно остановить)

```bash
# Остановить reconcile
flux suspend helmrelease service-py -n infra

# Возобновить
flux resume helmrelease service-py -n infra
```

---

## 4. HelmRelease завис в Progressing или Failed

### Симптом
```
NAME        READY   STATUS
service-py  False   Helm upgrade failed: ...
```

### Диагностика

```bash
# Подробный статус
kubectl describe helmrelease service-py -n infra

# История Helm релизов
helm history service-py -n infra

# Что сейчас задеплоено (живые values)
helm get values service-py -n infra
```

### Rollback через Helm

```bash
# Откатить на предыдущую ревизию
helm rollback service-py -n infra

# Откатить на конкретную ревизию
helm rollback service-py 1 -n infra
```

### Форс-пересоздание если Helm state сломан

```bash
# Удаляем HelmRelease — Flux пересоздаст
kubectl delete helmrelease service-py -n infra

# Через 1 минуту Flux сам применит его заново из Git
flux reconcile kustomization flux-system -n flux-system
```

---

## 5. service-py недоступен

**Стек:** Python/Flask → PostgreSQL → Redis → Kafka

### Диагностика

```bash
# Под живой?
kubectl get pod -l app=service-py -n infra

# Эндпоинт отвечает?
kubectl exec -it deploy/service-py -n infra -- curl -s localhost:8080/healthz

# Postgres доступен из пода?
kubectl exec -it deploy/service-py -n infra -- \
  python3 -c "import psycopg2; psycopg2.connect('host=postgres.infra.svc.cluster.local port=5432 dbname=orders user=postgres password=postgres'); print('OK')"

# Redis доступен?
kubectl exec -it deploy/service-py -n infra -- \
  python3 -c "import redis; r=redis.Redis(host='redis.infra.svc.cluster.local'); print(r.ping())"

# Kafka доступен?
kubectl exec -it deploy/service-py -n infra -- \
  curl -s kafka.infra.svc.cluster.local:9092

# Логи сервиса
kubectl logs -l app=service-py -n infra --tail=50 -f
```

### Проверка зависимостей

```bash
# Postgres
kubectl get pod -l app=postgres -n infra
kubectl logs -l app=postgres -n infra --tail=20

# Redis
kubectl get pod -l app=redis -n infra

# Kafka
kubectl get pod -l app=kafka -n infra
kubectl logs -l app=kafka -n infra --tail=20
```

---

## 6. service-go недоступен

**Стек:** Go/Gin → MongoDB → Redis → Kafka (consumer) → RabbitMQ

### Диагностика

```bash
# Под живой?
kubectl get pod -l app=service-go -n infra

# Healthcheck
kubectl exec -it deploy/service-go -n infra -- wget -qO- localhost:8080/healthz

# MongoDB доступен?
kubectl exec -it deploy/service-go -n infra -- \
  wget -qO- mongo.infra.svc.cluster.local:27017

# RabbitMQ доступен?
kubectl exec -it deploy/service-go -n infra -- \
  wget -qO- rabbitmq.infra.svc.cluster.local:15672/api/overview

# Логи сервиса
kubectl logs -l app=service-go -n infra --tail=50 -f
```

### Kafka consumer проверка

```bash
# Логи — ищем ошибки консюминга
kubectl logs -l app=service-go -n infra | grep -i "kafka\|consumer\|error"

# Kafka брокер живой?
kubectl get pod -l app=kafka -n infra
kubectl exec -it deploy/kafka -n infra -- \
  kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

## 7. Kafka / RabbitMQ / Redis / Mongo / Postgres упали

Универсальный сценарий для любого брокера или БД.

```bash
# Заменить APP на: kafka / rabbitmq / redis / mongo / postgres
APP=kafka

# Статус пода
kubectl get pod -l app=$APP -n infra -o wide

# Events — почему упал
kubectl describe pod -l app=$APP -n infra

# Логи
kubectl logs -l app=$APP -n infra --tail=50

# Рестарт (если завис)
kubectl rollout restart deployment/$APP -n infra

# Ждём готовности
kubectl rollout status deployment/$APP -n infra
```

---

## 8. OOMKilled — под убит по памяти

### Симптом
```
Last State: Terminated  Reason: OOMKilled
```

### Диагностика

```bash
# Смотрим текущие лимиты
kubectl get pod -l app=service-py -n infra \
  -o jsonpath='{.items[0].spec.containers[0].resources}' | python3 -m json.tool

# Смотрим потребление прямо сейчас
kubectl top pod -n infra

# Смотрим потребление ноды
kubectl top node
```

### Фикс — увеличить лимиты в values.yaml

В `infra.git` в `clusters/my-cluster/helm/service-py.yaml` добавить:

```yaml
values:
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

Закоммитить → Flux применит автоматически через 1 минуту.

---

## 9. Node под давлением (DiskPressure / MemoryPressure)

### Диагностика

```bash
# Условия ноды
kubectl describe node onbording-vm01 | grep -A10 Conditions

# Топ потребителей памяти
kubectl top pod -A --sort-by=memory | head -15

# Топ потребителей CPU
kubectl top pod -A --sort-by=cpu | head -15

# Диск на ноде (через SSH)
ssh onbording-vm01 "df -h && du -sh /var/lib/docker/* 2>/dev/null | sort -rh | head -10"
```

### Чистка образов на ноде

```bash
ssh onbording-vm01 "crictl rmi --prune"
```

---

## 10. OpenSearch не принимает логи

### Диагностика

```bash
# Fluent-Bit собирает логи?
kubectl logs -l app.kubernetes.io/name=fluent-bit -n logging --tail=30

# Ошибки отправки в OpenSearch
kubectl logs -l app.kubernetes.io/name=fluent-bit -n logging | grep -i "error\|failed\|retry"

# OpenSearch под живой?
kubectl get pod -n logging

# OpenSearch health (HTTPS с самоподписанным сертом)
kubectl exec -it opensearch-cluster-master-0 -n logging -- \
  curl -sk -u admin:admin https://localhost:9200/_cat/health?v

# Индексы
kubectl exec -it opensearch-cluster-master-0 -n logging -- \
  curl -sk -u admin:admin https://localhost:9200/_cat/indices?v
```

> **Важно:** OpenSearch слушает на `https://` с самоподписанным сертификатом.  
> Всегда используй флаги `-sk` и `-u admin:admin`

### Типичная проблема — HTTP вместо HTTPS

Fluent-Bit должен слать на `https://opensearch-cluster-master.logging.svc.cluster.local:9200`.  
Проверить конфиг:

```bash
kubectl get configmap -n logging -o yaml | grep -A5 "Host\|Port\|tls"
```

---

## 11. Vault не отдаёт секреты

### Диагностика

```bash
# Vault под живой?
kubectl get pod -n vault

# Vault статус (sealed?)
kubectl exec -it vault-0 -n vault -- vault status

# Логи Agent Injector
kubectl logs -l app.kubernetes.io/name=vault-agent-injector -n vault --tail=30

# Логи vault-agent-init в проблемном поде
kubectl logs -l app=service-py -n infra -c vault-agent-init

# Проверка секретов в Vault напрямую
kubectl exec -it vault-0 -n vault -- \
  vault kv get -mount=microservices service-py/dev
```

### Если Vault sealed (после рестарта в dev mode)

```bash
# В dev mode Vault сам не запечатывается при рестарте — но если случилось:
kubectl exec -it vault-0 -n vault -- vault operator unseal

# Проверить токен
kubectl exec -it vault-0 -n vault -- \
  sh -c 'VAULT_TOKEN=root vault kv list microservices/'
```

> **Напоминание:** Vault в dev mode — данные в памяти. После рестарта пода все секреты  
> нужно записывать заново. Для прода нужен Raft HA + auto-unseal.

---

## 12. Prometheus не скрейпит метрики

### Диагностика

```bash
# ServiceMonitor существует?
kubectl get servicemonitor -n infra

# Prometheus targets (через port-forward)
kubectl port-forward svc/kube-prometheus-stack-prometheus -n monitoring 9090:9090 &
# Открыть: http://localhost:9090/targets

# Логи Prometheus
kubectl logs -l app.kubernetes.io/name=prometheus -n monitoring --tail=30

# Alertmanager конфиг загружен?
kubectl exec -it alertmanager-kube-prometheus-stack-alertmanager-0 -n monitoring \
  -c alertmanager -- cat /etc/alertmanager/config_out/alertmanager.env.yaml
```

### Проверить что алерты срабатывают

```bash
# Активные алерты прямо сейчас
kubectl port-forward svc/kube-prometheus-stack-alertmanager -n monitoring 9093:9093 &
# Открыть: http://localhost:9093/#/alerts
```

---

## 13. Полный rollback сервиса

Используем когда новый деплой сломал сервис и нужно срочно вернуться назад.

```bash
# Шаг 1 — смотрим историю
helm history service-py -n infra

# Шаг 2 — откатываем на предыдущую ревизию
helm rollback service-py -n infra

# Шаг 3 — проверяем что откатилось
kubectl rollout status deployment/service-py -n infra
kubectl get pod -l app=service-py -n infra

# Шаг 4 — чтобы Flux не перезаписал rollback — suspend на время разбора
flux suspend helmrelease service-py -n infra

# После фикса в Git — resume
flux resume helmrelease service-py -n infra
```

---

## Шпаргалка — самые частые команды

```bash
# Общий статус кластера
kubectl get pods -A | grep -v Running

# Форс-reconcile всего
flux reconcile source git flux-system && flux reconcile kustomization flux-system

# Рестарт сервиса
kubectl rollout restart deployment/service-py -n infra
kubectl rollout restart deployment/service-go -n infra

# Логи в реальном времени
kubectl logs -l app=service-py -n infra -f --tail=50
kubectl logs -l app=service-go -n infra -f --tail=50

# Потребление ресурсов
kubectl top pod -n infra
kubectl top node

# Rollback
helm rollback service-py -n infra
helm rollback service-go -n infra
```

---

*Runbook построен на основе живого кластера `onbording-vm01` — все команды проверены.*  
*Последнее обновление: апрель 2026*
