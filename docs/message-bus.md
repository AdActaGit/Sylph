# Шина сообщений (Sylph.MessageBus)

Ядро Sylph. Даёт единый API `IMessageBus` для двух паттернов обмена и прозрачно
маршрутизирует вызовы локально (in-process) или удалённо (через транспорт).

## IMessageBus

```csharp
public interface IMessageBus
{
    // Запрос/ответ: один запрос → один ответ
    Task<TResponse> Request<TRequest, TResponse>(
        TRequest request,
        MessageBusHeaders? headers = null,
        CancellationToken cancellationToken = default)
        where TRequest : class, IRequest<TResponse>
        where TResponse : IResponse, new();

    // Публикация: уведомление → 0..N подписчиков
    Task Publish<TMessage>(
        TMessage message,
        MessageBusHeaders? headers = null,
        CancellationToken ct = default)
        where TMessage : class, INotify;
}
```

## Контракты

| Интерфейс | Роль |
|---|---|
| `IRequest<TResponse>` | маркер сообщения-запроса |
| `IResponse` (`new()`) | ответ; обязателен флаг `IsError` |
| `IRequestHandler<TRequest, TResponse>` | обработчик запроса (один на тип) |
| `INotify` | маркер уведомления |
| `INotifyHandler<TNotify>` | подписчик уведомления (много на тип) |

### Запрос/ответ

```csharp
// контракт
public record Customer
{
    public record LoyaltyRequest : IRequest<LoyaltyResponse>
    {
        public long CustomerId { get; set; }
        public decimal Amount { get; set; }
    }

    public record LoyaltyResponse : ResponseBase   // ResponseBase : IResponse
    {
        public string Tier { get; set; } = "";
        public decimal Discount { get; set; }
    }
}

// обработчик (реальный пример из SylphDemo.Customers)
public class CustomerLoyaltyHandler : IRequestHandler<Customer.LoyaltyRequest, Customer.LoyaltyResponse>
{
    public async Task<Customer.LoyaltyResponse> OnHandle(Customer.LoyaltyRequest request, CancellationToken ct)
    {
        await Task.Delay(TimeSpan.FromMilliseconds(50), ct);
        var tier = request.CustomerId % 3 == 0 ? "Gold" : "Standard";
        return new Customer.LoyaltyResponse
        {
            Tier = tier,
            Discount = tier == "Gold" ? Math.Round(request.Amount * 0.1m, 2) : 0,
        };
    }
}
```

Вызов из другого обработчика (агрегация сервисов — реальный `OrderCreateHandler`):

```csharp
public class OrderCreateHandler(IMessageBus messageBus)
    : IRequestHandler<Order.CreateRequest, Order.CreateResponse>
{
    public async Task<Order.CreateResponse> OnHandle(Order.CreateRequest request, CancellationToken ct)
    {
        var trace = await messageBus.Request<Order.CheckoutTraceRequest, Order.CheckoutTraceResponse>(
            new Order.CheckoutTraceRequest
            {
                CustomerId = request.CustomerId,
                ProductId  = request.ProductId,
                Quantity   = request.Quantity
            },
            cancellationToken: ct);

        return new Order.CreateResponse
        {
            OrderId = trace.OrderId,
            Status  = trace.IsError ? "Failed" : "Created",
            IsError = trace.IsError
        };
    }
}
```

### Публикация/подписка

```csharp
public record OrderShipped(long OrderId) : INotify;

public class EmailOnShipped : INotifyHandler<OrderShipped>
{
    public Task OnHandle(OrderShipped notify, CancellationToken ct)
    {
        // отправить письмо …
        return Task.CompletedTask;
    }
}

// где-то в коде
await bus.Publish(new OrderShipped(orderId));
```

## Регистрация

```csharp
using Sylph.MessageBus;

builder.Services.AddSylphMessageBus(options =>
{
    options.MaxParallelMessages          = Environment.ProcessorCount; // параллелизм обработки
    options.IncomingQueueCapacity        = 10_000;                     // ёмкость входной очереди
    options.StatisticPageRefreshInterval = TimeSpan.FromSeconds(15);   // период обновления UI
    options.DefaultHandlerVersion        = DefaultHandlerVersion.Hi;   // выбор версии по умолчанию
});

// автоскан обработчиков в указанных сборках
builder.Services.AddSylphMessageHandlers(Assembly.GetExecutingAssembly());
```

`AddSylphMessageHandlers` находит все классы, реализующие `IRequestHandler<,>`
или `INotifyHandler<>`, регистрирует их как `Scoped` и заполняет реестр
`MessageBusHandlerCollection` (он питает диагностический UI).

Ядро добавляет: `MessageBusRuntime` (hosted-сервис, обрабатывает входную очередь),
балансировку (`AddSylphMessageBalancing`), метрики, аксессоры трейс-контекста и
`NoopMessageBusTransport` (если не подключён реальный транспорт — работает только локально).

## Версии и режимы доставки

Обработчик можно пометить атрибутом:

```csharp
using Sylph.MessageBus.Contracts;

[RequestDelivery(RequestDeliveryMode.PreferLocal, "2.0.0.0")]
public class OrderCreateHandlerV2 : IRequestHandler<Order.CreateRequest, Order.CreateResponse>
{
    // …
}
```

- **Версия** (`"2.0.0.0"`) — позволяет держать несколько версий обработчика одного
  запроса. Выбор версии по умолчанию задаётся `MessageBusOptions.DefaultHandlerVersion`
  (`Hi` — наибольшая, `Low` — наименьшая, `Custom` — `CustomDefaultHandlerVersion`).
  Конкретную версию можно запросить через `MessageBusHeaders.RequestHandlerVersion`.
- **Режим доставки** (`RequestDeliveryMode`):
  - `PreferLocal` *(по умолчанию)* — локально, иначе через транспорт;
  - `LocalOnly` — только локально;
  - `RemoteOnly` — всегда через транспорт.

## Балансировка и метрики узлов

```csharp
builder.Services.AddSylphMicroserviceMetrics(
    configureMetrics: m => m.ReportInterval = TimeSpan.FromSeconds(5),
    configurePriorities: p =>      // веса метрик при выборе узла
    {
        p.Cpu = 1; p.Memory = 1; p.DiskSpace = 0.5; p.DiskUsage = 0.5; p.QueueLength = 2;
    },
    configureLimits: l =>          // лимиты, после которых узел считается перегруженным
    {
        l.MaxCpuUsagePercent       = 95;
        l.MinMemoryAvailableBytes  = 128L * 1024 * 1024;
        l.MaxQueueLength           = 10_000;
    });
```

Узлы регистрируются в `IMicroserviceNodeRegistry`, периодически репортят
`MicroserviceMetricsReported`, а `WeightedMicroserviceNodeSelector` выбирает наименее
загруженный узел. Состояние видно на [странице балансировщика](ui.md#страница-балансировщика).

## MessageBusHeaders

Заголовки, которыми можно управлять маршрутизацией и трейсингом запроса:

| Поле | Назначение |
|---|---|
| `Subject`, `Queue` | адресация в транспорте |
| `TargetNodeId` | принудительная отправка на конкретный узел |
| `RequestHandlerVersion` | выбор версии обработчика |
| `RequestDeliveryMode` | переопределение режима доставки для конкретного вызова |
| `CorrelationKey`, `MessageId` | корреляция |
| `WorkflowEventName`, `SuppressWorkflowBridge` | интеграция с воркфлоу |
| `TraceId`, `SpanId`, `ParentSpanId`, `TraceFlags`, `TraceState` | W3C-трейс |

## Наблюдаемость

`AddSylphOpenTelemetry("service-name")` подключает экспорт метрик/трейсов (OTLP).
Метрики шины: `SylphMessageBusMetrics`, балансировщика: `SylphBalancerMetrics`.
