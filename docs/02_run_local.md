# Run Local

## Overview

Инструкция по запуску сервисов локально через docker-compose для разработки.

## Prerequisites

- Docker Desktop установлен
- docker-compose установлен
- Git установлен

## Quick Start
```bash
# Клонировать репозиторий
git clone https://github.com/bogdanzemly-va/app.git
cd app

# Запустить все сервисы
docker-compose up -d

# Проверить что всё запустилось
docker-compose ps
```

## Services

| Сервис | Порт | URL |
|--------|------|-----|
| service-py | 8080 | http://localhost:8080 |
| service-go | 8081 | http://localhost:8081 |
| PostgreSQL | 5432 | localhost:5432 |
| MongoDB | 27017 | localhost:27017 |
| Redis | 6379 | localhost:6379 |
| Kafka | 9092 | localhost:9092 |
| RabbitMQ | 5672 | localhost:5672 |
| RabbitMQ UI | 15672 | http://localhost:15672 |

## Test API
```bash
# Создать заказ
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"customer": "test", "amount": 100}'

# Получить заказ
curl http://localhost:8080/orders/1

# Получить отгрузку
curl http://localhost:8081/shipments/1
```

## Stop
```bash
docker-compose down
```