# Быстрый старт

## Требования

- .NET SDK **10.0**
- (опционально) запущенный **NATS** для межсервисного обмена: `nats://localhost:4222`
- (опционально) **.NET Aspire** workload — для запуска всего демо-стека одной командой

## Сборка

```bash
cd src
dotnet build Sylph.slnx
```

## Минимальный сервис с обработчиком запроса

### 1. Контракт

```csharp
using Sylph.MessageBus.Contracts;

public record Ping
{
    public record Request : IRequest<Response>
    {
        public string Text { get; set; } = "";
    }

    public record Response : IResponse
    {
        public string Echo { get; set; } = "";
        public bool IsError { get; set; }
    }
}
```

> `IResponse` требует `IsError` и публичный конструктор без параметров (`new()`).
> В демо для этого есть базовый тип `ResponseBase`.

### 2. Обработчик

```csharp
using Sylph.MessageBus.Contracts;

public class PingHandler : IRequestHandler<Ping.Request, Ping.Response>
{
    public Task<Ping.Response> OnHandle(Ping.Request request, CancellationToken ct)
        => Task.FromResult(new Ping.Response { Echo = request.Text });
}
```

### 3. Регистрация и запуск

```csharp
using System.Reflection;
using Sylph.MessageBus;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSylphMessageBus();                                  // ядро шины
builder.Services.AddSylphMessageHandlers(Assembly.GetExecutingAssembly()); // скан обработчиков

var app = builder.Build();

app.MapGet("/ping", async (IMessageBus bus, string text) =>
    await bus.Request<Ping.Request, Ping.Response>(new Ping.Request { Text = text }));

app.Run();
```

Готово: `GET /ping?text=hello` вернёт `{"echo":"hello","isError":false}`.
Обработчик вызывается **локально** (in-process), транспорт не нужен.

## Добавляем диагностический UI

```csharp
using Sylph.MessageBus;

// после builder.Services.AddSylphMessageBus(...)
builder.Services.AddSylphMicroserviceMetrics(opt => opt.ReportInterval = TimeSpan.FromSeconds(5));

var app = builder.Build();
app.MapMessageBusStatisticPage("/");      // страница шины
app.MapBalancerStatisticPage("/balancer"); // страница балансировщика
app.Run();
```

Откройте `/` — увидите [страницу статистики шины](ui.md#страница-шины-сообщений).

## Межсервисный обмен через NATS

Чтобы запрос уходил на другой узел, подключите транспорт:

```csharp
using Sylph.MessageBus.Transport.NATS;

builder.Services.AddSylphNatsTransport(opt =>
{
    opt.Url = "nats://localhost:4222";
    opt.RequestTimeout = TimeSpan.FromSeconds(30);
});
```

Теперь если локального обработчика нет (режим `PreferLocal`/`RemoteOnly`),
запрос будет доставлен через NATS на узел, где обработчик есть.
См. [транспорты](transports.md) и [режимы доставки](message-bus.md#версии-и-режимы-доставки).

## Вариант А — запуск отдельного демо-сервиса

```bash
cd src/Samples/SylphDemo.Orders
dotnet run
# UI воркфлоу/шины поднимется на порту из Properties/launchSettings.json
```

## Вариант Б — весь стек через Aspire

`Sylph.Aspire` поднимает Gateway + все микросервисы (Orders, Products×2, Payment,
Warehouse, Customers) + песочницу воркфлоу, прокидывает `NATS_URL` и OTLP-экспорт.

```bash
cd src/Sylph.Aspire
dotnet run
```

Предварительно поднимите NATS (по умолчанию `nats://localhost:4222`) — он подключён
как внешний сервис в `AppHost.cs`. Дашборд Aspire покажет все ресурсы и их эндпоинты;
шлюз слушает `http://localhost:5182`, песочница воркфлоу — `http://localhost:5190`.

## Песочница воркфлоу

```bash
cd src/Samples/WebApplicationWFPlay
dotnet run --no-launch-profile
# http://localhost:5079
```

Полезные эндпоинты песочницы (запускают демо-саги и наполняют UI данными):

| Эндпоинт | Что делает |
|---|---|
| `GET /run_wf` | синхронно выполняет `QuickDocumentQualityWorkflow` |
| `GET /run_onboarding_wf?express=&training=&complianceBlock=` | онбординг сотрудника (синхронно) |
| `GET /start_onboarding_wf` | запускает онбординг асинхронно (вернёт `executionId`) |
| `GET /start_review_wf` | запускает сагу ручного ревью (ждёт события) |
| `POST /complete_review_wf/{reviewId}?approved=true` | присылает событие-решение |
| `GET /` | [Workflow UI](ui.md) |
| `GET /mb` | [MessageBus UI](ui.md#страница-шины-сообщений) |
