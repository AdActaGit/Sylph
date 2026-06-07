# Документация Sylph

**Sylph** — это набор .NET-библиотек для построения распределённых систем на микросервисах:
лёгкая шина сообщений (in-process + транспорт), HTTP-шлюз, движок саг/воркфлоу и
встроенный диагностический UI. Всё работает на `net10.0`.

> Название отсылает к *сильфам* — духам стихии воздуха: невидимым, лёгким и подвижным
> (см. [корневой README](../README.md)).

## Состав решения

| Область | Проект | Назначение |
|---|---|---|
| **Шина сообщений** | `Sylph.MessageBus` | Ядро: `IMessageBus`, обработчики `IRequestHandler`/`INotifyHandler`, балансировка, метрики, трейсинг |
| | `Sylph.MessageBus.Gateway` | HTTP-шлюз: генерация REST-эндпоинтов из контрактов + Swagger |
| | `Sylph.MessageBus.Transport.NATS` | Транспорт поверх NATS (request/reply, pub/sub) |
| | `Sylph.MessageBus.Transport.Kafka` | Транспорт поверх Kafka |
| | `Sylph.MessageBus.Transport.RabbitMQ` | Транспорт поверх RabbitMQ |
| **Воркфлоу** | `Sylph.Workflow` | Движок саг: шаги, ветвления, компенсации, wait-event, delay, outbox, recovery |
| | `Sylph.Workflow.Quartz` | Планировщик на Quartz |
| | `Sylph.Workflow.Store.{Mssql,Postgres,Redis}` | Персистентные хранилища состояния воркфлоу |
| **UI** | `Sylph.UI` | Диагностические страницы шины и балансировщика |
| | `Sylph.Workflow.UI` | Диагностический UI воркфлоу: статистика, графы, block-flow |
| **Хостинг** | `Sylph.Aspire` | .NET Aspire AppHost для локального запуска всего стека |
| **Примеры** | `SylphDemo.*` | Демо-маркетплейс (Orders, Products, Payment, Warehouse, Customers, Gateway) |
| | `WebApplicationWFPlay` | Песочница воркфлоу |

## Оглавление

1. [Быстрый старт](getting-started.md) — поднять первый сервис за 5 минут
2. [Архитектура](architecture.md) — как устроены и связаны компоненты
3. [Шина сообщений](message-bus.md) — `IMessageBus`, обработчики, балансировка, версии
4. [HTTP-шлюз](gateway.md) — REST из контрактов, Swagger, трейсинг
5. [Транспорты](transports.md) — NATS / Kafka / RabbitMQ
6. [Воркфлоу](workflow.md) — саги, компенсации, wait-event, delay, хранилища
7. [Диагностический UI](ui.md) — страницы и скриншоты

## С чего начать

- Хотите попробовать локально → [Быстрый старт](getting-started.md)
- Хотите понять модель → [Архитектура](architecture.md)
- Пишете обработчик запроса → [Шина сообщений](message-bus.md)
- Пишете сагу → [Воркфлоу](workflow.md)
