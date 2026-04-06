# Architecture

## Overview

Проект состоит из двух микросервисов которые обрабатывают заказы и отгрузки.

## Components
```mermaid
graph TD
    Client -->|POST /orders| ServicePy
    Client -->|GET /orders/id| ServicePy
    Client -->|POST /shipments| ServiceGo
    Client -->|GET /shipments/id| ServiceGo

    ServicePy -->|read/write| PostgreSQL
    ServicePy -->|cache| Redis
    ServicePy -->|publish order.created| Kafka

    Kafka -->|consume order.created| ServiceGo
    ServiceGo -->|read/write| MongoDB
    ServiceGo -->|cache| Redis
    ServiceGo -->|publish shipment.ready| RabbitMQ

    ServicePy -.->|secrets| Vault
    ServiceGo -.->|secrets| Vault

    Prometheus -->|scrape metrics| ServicePy
    Prometheus -->|scrape metrics| ServiceGo
    Grafana -->|visualize| Prometheus
```

## Services

**Service-Py** (Python/Flask) — управление заказами
- `POST /orders` — создать заказ
- `GET /orders/{id}` — получить заказ
- База данных: PostgreSQL
- Кэш: Redis (60 сек)
- Отправляет событие `order.created` в Kafka

**Service-Go** (Go/Gin) — управление отгрузками
- `POST /shipments` — создать отгрузку
- `GET /shipments/{order_id}` — получить отгрузку
- База данных: MongoDB
- Кэш: Redis (30 сек)
- Читает `order.created` из Kafka
- Отправляет `shipment.ready` в RabbitMQ

## Infrastructure

| Компонент | Назначение | Namespace |
|-----------|-----------|-----------|
| PostgreSQL | База данных заказов | infra |
| MongoDB | База данных отгрузок | infra |
| Redis | Кэш для обоих сервисов | infra |
| Kafka | Очередь событий | infra |
| RabbitMQ | Брокер сообщений | infra |
| Vault | Управление секретами | vault |
| Prometheus | Сбор метрик | monitoring |
| Grafana | Визуализация метрик | monitoring |
| FluxCD | GitOps деплой | flux-system |
