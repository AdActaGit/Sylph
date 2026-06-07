# Воркфлоу / Саги (Sylph.Workflow)

Движок длительных процессов с состоянием: декларативное описание шагов, ветвления,
компенсации (saga pattern), ожидание внешних событий, задержки, transactional outbox,
автоматическое восстановление «зависших» исполнений и сменные хранилища состояния.

## Базовые понятия

| Понятие | Тип | Описание |
|---|---|---|
| Воркфлоу | `Workflow<TInput, TOutput, TContext>` | описание процесса (наследуете и переопределяете `Configure`) |
| Контекст | `WorkflowContext<TInput, TOutput>` | состояние исполнения, сериализуется в хранилище |
| Шаг | `IWorkflowStep<TContext>` | единица работы; возвращает `WorkflowStepResult` |
| Компенсация | `IWorkflowCompensationStep<TContext>` | откат шага при ошибке |
| Ожидание события | `WaitEventStep<TContext, TEvent>` | приостановка до внешнего события (`INotify`) |
| Движок | `IWorkflowEngine` | запуск (`Run`/`Start`), события (`PublishEvent`), снимки |
| Реестр | `IWorkflowRegistry` | все зарегистрированные воркфлоу (питает UI/графы) |

## Описание воркфлоу

`Configure()` строится fluent-билдером. Доступные конструкции (`Workflow<,,>`):

| Конструкция | Назначение |
|---|---|
| `StartWith<TStep>()` | стартовый шаг |
| `Step<TStep>(cfg => …)` | следующий шаг (autolink от предыдущего top-level) |
| `Delay(TimeSpan)` / `Delay<TDynamicStep>()` | фиксированная / вычисляемая задержка |
| `WaitEvent<TStep, TEvent>(cfg => …)` | ждать внешнее событие |

Конфигурация шага (`cfg`):

| Метод | Назначение |
|---|---|
| `.NextStepIf<TStep>(ctx => условие)` | условный переход (ветвление) |
| `.OnError<TStep>()` | переход при ошибке шага |
| `.WithCompensation<TComp>(cmp => cmp.Retry(n))` | компенсация с ретраями |
| `.MaxEntryCount(n)` | макс. число входов в шаг (защита от циклов) |
| `.Retry(n)` | ретраи самого шага |

### Результат шага — `WorkflowStepResult`

```csharp
WorkflowStepResult.Success();                 // успех → переход дальше
WorkflowStepResult.Failed("error.code");      // ошибка → OnError/компенсация
WorkflowStepResult.Skipped();                 // пропустить
WorkflowStepResult.Stop();                    // остановить воркфлоу
// доставка сообщений через outbox:
WorkflowStepResult.Success().WithOutbox(msg1, msg2);
```

## Полный пример (онбординг сотрудника)

Реальный `EmployeeOnboardingWorkflow` из `WebApplicationWFPlay` — 23 узла: ветвления,
компенсации, wait-event с таймаутом и delay. Так выглядит его описание:

```csharp
public sealed class EmployeeOnboardingWorkflow
    : Workflow<EmployeeOnboardingInput, EmployeeOnboardingOutput, EmployeeOnboardingContext>
{
    protected override EmployeeOnboardingContext CreateContext(EmployeeOnboardingInput input)
        => new() { Input = input };

    protected override void Configure()
    {
        StartWith<ValidateOnboardingRequestStep>();

        Step<CreateEmployeeDraftStep>(cfg => cfg
            .WithCompensation<UndoCreateEmployeeDraftCompensation>(cmp => cmp.Retry(2)));

        // ветвление: экспресс или стандартный путь
        Step<CheckOnboardingModeStep>(cfg => cfg
            .NextStepIf<ExpressBackgroundCheckStep>(ctx => ctx.Input.IsExpress)
            .NextStepIf<StandardBackgroundCheckStep>(ctx => !ctx.Input.IsExpress));

        Step<ExpressBackgroundCheckStep>();
        Step<StandardBackgroundCheckStep>();

        Step<AllocateWorkspaceStep>(cfg => cfg
            .WithCompensation<ReleaseWorkspaceCompensation>());

        Step<ProvisionAccountsStep>(cfg => cfg
            .MaxEntryCount(3)
            .Retry(2)
            .OnError<ProvisionAccountsFailedStep>()
            .WithCompensation<DeprovisionAccountsCompensation>());

        Step<CheckComplianceStep>(cfg => cfg
            .NextStepIf<RejectOnboardingStep>(ctx => ctx.ComplianceBlocked)
            .NextStepIf<WaitManagerApprovalStep>(ctx => ctx.RequiresManagerApproval)
            .NextStepIf<AssignEquipmentStep>(ctx => !ctx.RequiresManagerApproval));

        // ожидание внешнего решения менеджера с таймаутом 5 минут
        WaitEvent<WaitManagerApprovalStep, ManagerApprovalCompletedEvent>(evt =>
        {
            evt.CorrelateBy(ctx => ctx.Input.OnboardingId.ToString("N"));
            evt.TimeoutAfter(TimeSpan.FromMinutes(5));
            evt.OnTimeout<ManagerApprovalTimeoutStep>();
        });

        Step<AssignEquipmentStep>(cfg => cfg.WithCompensation<ReturnEquipmentCompensation>());

        Delay(TimeSpan.FromSeconds(3));

        Step<SendWelcomeNotificationStep>();

        Step<CheckTrainingRequiredStep>(cfg => cfg
            .NextStepIf<ScheduleTrainingStep>(ctx => ctx.Input.RequiresTraining)
            .NextStepIf<RegisterInHrSystemStep>(ctx => !ctx.Input.RequiresTraining));

        Step<ScheduleTrainingStep>();
        Step<RegisterInHrSystemStep>();
        Step<CompleteOnboardingStep>();
    }
}
```

Граф этого воркфлоу в UI:

![Block-flow онбординга](images/ui-workflow-blockflow.png)

### Input / Output / Context

```csharp
public sealed record EmployeeOnboardingInput(
    Guid OnboardingId, string EmployeeEmail, string Department,
    bool IsExpress, bool RequiresTraining, bool SimulateComplianceBlock);

public sealed record EmployeeOnboardingOutput(
    Guid OnboardingId, Guid EmployeeId, string Status, IReadOnlyList<string> Trace);

public sealed class EmployeeOnboardingContext
    : WorkflowContext<EmployeeOnboardingInput, EmployeeOnboardingOutput>
{
    public Guid EmployeeId { get; set; }
    public bool ComplianceBlocked { get; set; }
    public bool RequiresManagerApproval { get; set; }
    public List<string> Trace { get; } = [];
}
```

### Шаг

Описание шага можно снабдить `[StepDescription(...)]` — текст попадёт в карточку UI:

```csharp
[StepDescription("Validates the onboarding request and required employee email.")]
public sealed class ValidateOnboardingRequestStep : IWorkflowStep<EmployeeOnboardingContext>
{
    public Task<WorkflowStepResult> Execute(EmployeeOnboardingContext ctx, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(ctx.Input.EmployeeEmail))
            return Task.FromResult(WorkflowStepResult.Failed("onboarding.email.required"));

        ctx.Trace.Add("validated");
        return Task.FromResult(WorkflowStepResult.Success());
    }
}
```

### Компенсация

```csharp
public sealed class ReleaseWorkspaceCompensation : IWorkflowCompensationStep<EmployeeOnboardingContext>
{
    public Task<WorkflowStepResult> Compensate(
        EmployeeOnboardingContext ctx,
        IReadOnlyList<WorkflowStepExecutionFrame> frames,
        CancellationToken ct)
    {
        ctx.WorkspaceAllocated = false;
        ctx.Trace.Add("compensation-workspace-released");
        return Task.FromResult(WorkflowStepResult.Success());
    }
}
```

При ошибке шага без `OnError` движок выполняет компенсации **уже пройденных** шагов
в обратном порядке (saga rollback).

### Ожидание внешнего события

Событие — это `INotify`, который связывается с исполнением по корреляционному ключу:

```csharp
public sealed record ManagerApprovalCompletedEvent(Guid OnboardingId, bool Approved, string Reviewer)
    : INotify, IWorkflowCorrelatedEvent
{
    public string WorkflowCorrelationKey => OnboardingId.ToString("N");
}

public sealed class WaitManagerApprovalStep
    : WaitEventStep<EmployeeOnboardingContext, ManagerApprovalCompletedEvent>
{
    public override Task<WorkflowStepResult> OnEvent(
        EmployeeOnboardingContext ctx,
        WorkflowEventEnvelope<ManagerApprovalCompletedEvent> envelope,
        CancellationToken ct)
    {
        ctx.ManagerApproved = envelope.Payload.Approved;
        return Task.FromResult(WorkflowStepResult.Success());
    }
}
```

Доставка события в движок:

```csharp
await workflowEngine.PublishEvent(new WorkflowEventEnvelope<ManagerApprovalCompletedEvent>
{
    EventId        = Guid.NewGuid().ToString("N"),
    EventName      = typeof(ManagerApprovalCompletedEvent).FullName!,
    CorrelationKey = payload.WorkflowCorrelationKey,
    Payload        = payload
});
```

## Запуск воркфлоу

```csharp
// Синхронно — ждём завершения (подходит, если нет долгих wait/delay)
var result = await engine.Run<MyWorkflow, MyInput, MyOutput, MyContext>(
    new MyInput(...),
    new WorkflowRunOptions { IdempotencyKey = Guid.NewGuid().ToString("N") });

// Асинхронно — получаем handle и executionId, процесс идёт в фоне
var handle = await engine.Start<MyWorkflow, MyInput, MyOutput, MyContext>(
    new MyInput(...),
    new WorkflowStartOptions { IdempotencyKey = id.ToString("N") });
// handle.ExecutionId, handle.Status, handle.WorkflowType, handle.WorkflowVersion
```

`IdempotencyKey` защищает от повторного запуска одного и того же процесса.

## Регистрация

```csharp
using Sylph.Workflow;
using Sylph.Workflow.UI;

builder.Services.AddSylphWorkflows(options =>
{
    options.DefaultMaxEntryCount            = 3;   // дефолтный лимит входов в шаг
    options.MaxTotalStepExecutions          = 100; // защита от бесконечных циклов
    options.MaxConcurrentBackgroundExecutions = 4; // параллелизм фоновых исполнений
}, Assembly.GetExecutingAssembly());               // скан воркфлоу и шагов

builder.Services.AddSylphWorkflowWorker();          // фоновые воркеры (worker/outbox/recovery)
builder.Services.AddSylphWorkflowMessageBusBridge();// мост outbox → шина сообщений
builder.Services.AddSylphWorkflowUi();              // регистрация UI

var app = builder.Build();
app.MapSylphWorkflowUi("/workflow");                // диагностический UI воркфлоу
app.Run();
```

`AddSylphWorkflows` валидирует каждое определение (`WorkflowDefinitionValidator`),
регистрирует воркфлоу, его шаги и компенсации в DI, а также in-memory реализации
хранилищ/очередей/локов.

Фоновые сервисы (`AddSylphWorkflowWorker`):

- `WorkflowBackgroundWorker` — исполняет асинхронно запущенные воркфлоу;
- `WorkflowOutboxDispatcher` — доставляет outbox-сообщения в шину;
- `WorkflowRecoveryWorker` — восстанавливает «зависшие»/таймаутнутые исполнения.

## Хранилища состояния

По умолчанию состояние хранится **в памяти** (`InMemoryWorkflowExecutionStore`).
Для продакшена подключите персистентное хранилище и укажите провайдера в конфигурации:

```jsonc
// appsettings.json
{
  "Sylph": { "Workflow": { "Store": { "Provider": "Postgres" /* | Mssql | Redis */ } } }
}
```

```csharp
using Sylph.Workflow;

// зарегистрировать доступных провайдеров (пример для Postgres/Redis)
builder.Services.AddPostgresWorkflowStore(/* строка подключения / опции */);
builder.Services.AddRedisWorkflowStore(/* … */);

// выбрать провайдера по конфигурации Sylph:Workflow:Store:Provider
builder.Services.AddSylphWorkflowStore(builder.Configuration);
```

| Провайдер | Пакет |
|---|---|
| MSSQL | `Sylph.Workflow.Store.Mssql` |
| PostgreSQL | `Sylph.Workflow.Store.Postgres` |
| Redis | `Sylph.Workflow.Store.Redis` |

Если провайдер не найден среди зарегистрированных — `WorkflowStoreProviderNotFoundException`.

## Планировщик Quartz

`Sylph.Workflow.Quartz` (`WorkflowQuartzExtensions`) позволяет запускать воркфлоу по
расписанию через Quartz — подключается дополнительно к основной регистрации.

## Диагностика

UI воркфлоу даёт статистику исполнений, граф определений и block-flow каждого процесса —
см. [ui.md](ui.md#страницы-воркфлоу). API (под выбранным префиксом, напр. `/workflow`):
`/api/workflows`, `/api/executions`, `/api/waiting`, `/api/dead-letters`,
`/api/executions/{id}` и ручные операции (retry, resume-from-step, mark-resolved,
dead-letter, update-context).
