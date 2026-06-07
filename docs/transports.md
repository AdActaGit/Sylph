# Транспорты

Транспорт — это реализация `IMessageBusTransport`, через которую шина доставляет
сообщения **между процессами/узлами**. Без транспорта работает только локальная
обработка (`NoopMessageBusTransport`). Транспорт нужен для режимов доставки
`PreferLocal` (когда локального обработчика нет) и `RemoteOnly`
(см. [режимы доставки](message-bus.md#версии-и-режимы-доставки)).

| Транспорт | Пакет | Метод подключения |
|---|---|---|
| **NATS** | `Sylph.MessageBus.Transport.NATS` | `AddSylphNatsTransport(...)` |
| **Kafka** | `Sylph.MessageBus.Transport.Kafka` | `AddSylphKafkaTransport()` |
| **RabbitMQ** | `Sylph.MessageBus.Transport.RabbitMQ` | `AddSylphRabbitMqTransport()` |

> Подключайте **один** транспорт на процесс. NATS — основной и наиболее проработанный
> (поддерживает request/reply и подписки, используется во всех демо).

## NATS

Основной транспорт. Реализует request/reply и pub/sub, поднимает hosted-подписчика
`NatsMessageBusSubscriber`.

```csharp
using Sylph.MessageBus.Transport.NATS;

builder.Services.AddSylphNatsTransport(options =>
{
    options.Url           = "nats://localhost:4222"; // адрес сервера
    options.RequestTimeout = TimeSpan.FromSeconds(30); // таймаут request/reply
    options.ClientName     = "orders";                 // имя клиента (по умолчанию — имя приложения)
});
```

Если `ClientName` не задан, он выводится из `IHostEnvironment.ApplicationName`,
иначе — из имени входной сборки. Демо-сервисы берут адрес из переменной окружения
`NATS_URL` (её проставляет Aspire `AppHost`).

### Локальный запуск NATS

```bash
docker run --rm -p 4222:4222 -p 8222:8222 nats:latest
```

(`8222` — HTTP-мониторинг; в Aspire он подключён как `nats-monitor`.)

## Kafka

```csharp
using Sylph.MessageBus.Transport.Kafka;

builder.Services.AddSylphKafkaTransport();
```

Регистрирует `KafkaMessageBusTransport` как `IMessageBusTransport`.

## RabbitMQ

```csharp
using Sylph.MessageBus.Transport.RabbitMQ;

builder.Services.AddSylphRabbitMqTransport();
```

Регистрирует `RabbitMqMessageBusTransport` как `IMessageBusTransport`.

## Как шина выбирает путь доставки

```
bus.Request(...)
   │
   ├─ есть локальный обработчик и режим ≠ RemoteOnly ──▶ выполнить in-process
   │
   └─ иначе ──▶ IMessageBusTransport (NATS/Kafka/Rabbit) ──▶ узел с обработчиком
```

Целевой узел при удалённой доставке подсказывает балансировщик
(`MessageBusHeaders.TargetNodeId`), а сериализацию конверта обеспечивает транспорт
(`MessageBusEnvelope`). Подробнее о выборе узла — [балансировка](message-bus.md#балансировка-и-метрики-узлов).
