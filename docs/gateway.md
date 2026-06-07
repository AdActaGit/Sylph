# HTTP-шлюз (Sylph.MessageBus.Gateway)

Шлюз превращает контракты запросов в REST-эндпоинты: декорируете запрос атрибутом
`[GatewayEndpoint]` — и получаете HTTP-маршрут, который десериализует тело, выбирает
узел через балансировщик, отправляет запрос в шину и возвращает ответ. Плюс Swagger
и сквозной трейсинг «из коробки».

## Объявление эндпоинта

Атрибут вешается на тип запроса в сборке контрактов (реальный пример — `Order`):

```csharp
using Sylph.Gateway;
using Sylph.MessageBus.Contracts;

public record Order
{
    [GatewayEndpoint(GatewayHttpMethod.Post, "/order/create", Tag = "Order", Description = "Create order")]
    public record CreateRequest : IRequest<CreateResponse>
    {
        public long CustomerId { get; set; }
        public long ProductId  { get; set; }
        public int  Quantity   { get; set; } = 1;
    }

    public record CreateResponse : ResponseBase
    {
        public long   OrderId { get; set; }
        public string Status  { get; set; } = "";
    }
}
```

### Параметры `[GatewayEndpoint]`

| Параметр | Описание |
|---|---|
| `method` | HTTP-метод (`GatewayHttpMethod.Get/Post/...`) |
| `route` | путь маршрута, например `/order/create` |
| `Name` | имя эндпоинта (для роутинга/OpenAPI) |
| `Tag` | группа в Swagger |
| `Version` | версия обработчика, в которую уйдёт запрос |
| `Description` | описание для Swagger |
| `AllowAnonymous` | разрешить анонимный доступ |

Атрибут `AllowMultiple = true` — на один контракт можно повесить несколько маршрутов.

## Подключение

```csharp
using Sylph.Gateway;
using Sylph.MessageBus;
using Sylph.MessageBus.Transport.NATS;

var builder = WebApplication.CreateBuilder(args);

// сканирует сборку контрактов и регистрирует эндпоинты + Swagger
builder.Services.AddGatewayEndpoints(typeof(SylphDemo.Contracts.Product).Assembly);

builder.Services.AddSylphMessageBus(o => o.StatisticPageRefreshInterval = TimeSpan.FromSeconds(15));
builder.Services.AddSylphMicroserviceMetrics(m => m.ReportInterval = TimeSpan.FromSeconds(5));
builder.Services.AddSylphNatsTransport(o => o.RequestTimeout = TimeSpan.FromSeconds(30));

var app = builder.Build();

app.MapGatewayEndpoints();            // регистрирует HTTP-маршруты
app.UseGatewaySwagger();              // Swagger UI
app.MapGatewayStatisticPage("/");     // статистика шлюза
app.MapMessageBusStatisticPage("/message-bus");
app.MapBalancerStatisticPage("/balancer");

app.Run();
```

`AddGatewayEndpoints` также проверяет **дубликаты маршрутов** — два контракта с одним
`route` приведут к `InvalidOperationException` на старте.

## Что делает шлюз на каждый запрос

Последовательность в `HandleRequest` (`GatewayEndpointRouteBuilderExtensions`):

1. Инкремент статистики эндпоинта (`GatewayEndpointStatistics`).
2. Чтение входящего трейс-контекста (`traceparent` или `X-Trace-Id`), создание scope.
3. Десериализация тела в тип запроса (`application/json`).
4. Выбор узла через `IBalancer.SelectAsync(queue)`.
5. Формирование `MessageBusHeaders` (queue, targetNodeId, версия, trace).
6. `IMessageBus.Request(...)` (через reflection по типам запроса/ответа).
7. Возврат ответа и проброс `traceparent`/`X-Trace-Id` в заголовки ответа.

### Коды ошибок

| Ситуация | HTTP |
|---|---|
| Пустое/невалидное тело | `400 Bad Request` |
| Таймаут запроса в шине | `504 Gateway Timeout` |
| Сбой транспорта/обработчика | `503 Service Unavailable` |
| Успех | `200 OK` + тело ответа |

## Трейсинг

Шлюз — точка входа трассировки: создаёт серверный span, кладёт `traceparent` в заголовки
шины, а ответу добавляет `traceparent` и `X-Trace-Id`, чтобы клиент мог связать вызовы.
Полностью совместимо с OpenTelemetry (`AddSylphOpenTelemetry`).

## Демо-эндпоинты маркетплейса

Из `SylphDemo.Contracts` (теги `Order` и `Trace`):

| Метод | Маршрут | Назначение |
|---|---|---|
| POST | `/order/create` | создать заказ |
| POST | `/order/get` | получить заказ |
| POST | `/order/list` | список заказов клиента |
| POST | `/trace/checkout` | сквозной чек-аут через все сервисы |
| POST | `/trace/fanout` | параллельный fan-out по сервисам |
| POST | `/trace/compensation` | трасса с компенсацией (refund + release) |

Пример:

```bash
curl -X POST http://localhost:5182/trace/checkout \
  -H "Content-Type: application/json" \
  -d '{"customerId":42,"productId":1001,"quantity":2}'
```
