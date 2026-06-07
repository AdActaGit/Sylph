# Архитектура

## Общая картина

```
                    ┌─────────────────────────────────────────────┐
   HTTP-клиент ───▶ │  Gateway (Sylph.MessageBus.Gateway)          │
                    │  REST-эндпоинты из [GatewayEndpoint] +Swagger│
                    └───────────────┬─────────────────────────────┘
                                    │ IMessageBus.Request(...)
                                    ▼
        ┌───────────────────────────────────────────────────────────┐
        │  Шина сообщений (Sylph.MessageBus)                          │
        │  • IRequestHandler<TReq,TResp>  — запрос/ответ              │
        │  • INotifyHandler<TNotify>      — публикация/подписка       │
        │  • Балансировщик + метрики узлов (CPU/RAM/диск/очередь)     │
        │  • Трейсинг (W3C traceparent)                               │
        └───────┬───────────────────────────────────────┬───────────┘
                │ локально (PreferLocal/LocalOnly)        │ удалённо (RemoteOnly)
                ▼                                         ▼
       обработчик в этом процессе               ┌──────────────────────┐
                                                │ IMessageBusTransport  │
                                                │ NATS / Kafka / Rabbit │
                                                └──────────┬───────────┘
                                                           ▼
                                                  другой узел-микросервис

        ┌───────────────────────────────────────────────────────────┐
        │  Движок воркфлоу (Sylph.Workflow)                          │
        │  шаги · ветвления · компенсации · wait-event · delay        │
        │  outbox → шина · recovery · персистентные хранилища         │
        └───────────────────────────────────────────────────────────┘

        ┌───────────────────────────────────────────────────────────┐
        │  Диагностический UI (Sylph.UI / Sylph.Workflow.UI)         │
        └───────────────────────────────────────────────────────────┘
```

## Ключевые концепции

### 1. Сообщения и обработчики

Два вида взаимодействия:

- **Запрос/ответ** — `IRequest<TResponse>` + `IRequestHandler<TRequest, TResponse>`.
  Один запрос → ровно один ответ.
- **Уведомление** — `INotify` + `INotifyHandler<TNotify>`.
  Публикация → 0..N подписчиков (fire-and-forget).

Обработчики регистрируются сканированием сборок (`AddSylphMessageHandlers(assembly)`),
их интерфейсы определяются автоматически. Подробнее — [message-bus.md](message-bus.md).

### 2. Локальная vs удалённая доставка

`MessageBusRuntime` сначала ищет обработчик **в текущем процессе**. Поведение настраивается
атрибутом `[RequestDelivery(RequestDeliveryMode.…)]` на обработчике:

| Режим | Поведение |
|---|---|
| `PreferLocal` (по умолчанию) | Локально, если обработчик есть; иначе через транспорт |
| `LocalOnly` | Только локально |
| `RemoteOnly` | Всегда через транспорт (NATS/Kafka/Rabbit) |

Это позволяет писать один и тот же код, который в монолите работает in-process,
а при масштабировании — по сети, без изменения обработчиков.

### 3. Балансировка узлов

Каждый узел публикует метрики (CPU, память, диск, длина очереди) через
`AddSylphMicroserviceMetrics(...)`. `IBalancer` / `WeightedMicroserviceNodeSelector`
выбирают целевой узел по взвешенной оценке метрик (приоритеты и лимиты настраиваются).
Шлюз использует балансировщик, чтобы выбрать узел перед отправкой запроса.

### 4. Трейсинг

Сквозной трейс по стандарту W3C (`traceparent`). Шлюз читает входящий `traceparent`
(или `X-Trace-Id`), создаёт scope (`SylphActivity`) и прокидывает контекст в заголовки
шины (`MessageBusHeaders`), а оттуда — в транспорт. Совместимо с OpenTelemetry
(`AddSylphOpenTelemetry("service-name")`).

### 5. Воркфлоу (саги)

`Sylph.Workflow` — это движок длительных процессов с состоянием:

- декларативное описание (`StartWith`/`Step`/`Delay`/`WaitEvent`) — fluent-builder;
- **компенсации** (`WithCompensation`) для отката при ошибке (saga pattern);
- **ожидание внешних событий** (`WaitEvent`) с таймаутами;
- **outbox** — шаги могут поставить сообщения в outbox, диспетчер доставит их в шину;
- **recovery** — фоновые воркеры восстанавливают «зависшие» исполнения;
- сменные **хранилища** состояния: in-memory (по умолчанию), MSSQL, Postgres, Redis.

Подробнее — [workflow.md](workflow.md).

## Поток обработки HTTP-запроса (демо-маркет)

На примере `SylphDemo`:

```
POST /order/create                       (Gateway)
  └▶ IMessageBus.Request<Order.CreateRequest, Order.CreateResponse>
       └▶ OrderCreateHandler                       (микросервис Orders)
            └▶ Request<Order.CheckoutTraceRequest> (агрегирует другие сервисы)
                 ├▶ Customers  (лояльность/скидка)
                 ├▶ Products   (наличие/рекомендации)
                 ├▶ Warehouse  (резерв склада)
                 └▶ Payment    (авторизация платежа)
```

Каждый сервис — отдельный процесс; между ними сообщения идут через NATS-транспорт.
Локально весь стек поднимается через [.NET Aspire](getting-started.md#вариант-б--весь-стек-через-aspire).

## Карта зависимостей проектов

```
Sylph.MessageBus            ← ядро, без внешних зависимостей
  ├─ Sylph.MessageBus.Gateway
  ├─ Sylph.MessageBus.Transport.NATS
  ├─ Sylph.MessageBus.Transport.Kafka
  ├─ Sylph.MessageBus.Transport.RabbitMQ
  └─ Sylph.Workflow         ← движок воркфлоу зависит от контрактов шины
       ├─ Sylph.Workflow.Quartz
       ├─ Sylph.Workflow.Store.Mssql
       ├─ Sylph.Workflow.Store.Postgres
       └─ Sylph.Workflow.Store.Redis
Sylph.UI                    ← диагностика шины/балансировщика
Sylph.Workflow.UI           ← диагностика воркфлоу (зависит от Sylph.UI и Sylph.Workflow)
```
