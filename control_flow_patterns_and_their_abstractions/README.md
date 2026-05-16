# Отчёт: Три управленческих шаблона и абстракции для них

## Введение

В материале есть очень показательный момент: Хеттингер берёт 20 строк кода с **try-except-else-finally** вокруг сетевого ресурса и сворачивает их в три строки через **with**. И главное — объясняет, почему это вообще работает. Дело не в том, что одна конструкция «короче» другой. Дело в том, что *сам узор* «инициализировать ресурс → поработать → закрыть» — это шаблон, и его не должно быть видно в каждом отдельном месте использования.

Я пошёл искать такие шаблоны у себя в проекте — и нашёл три штуки, которые буквально просят, чтобы их обернули в абстракцию. Не классические GoF-паттерны, а именно повторяющиеся управляющие конструкции: «получи и проверь», «открой-поработай-закрой», «попробуй-сообщи-результат». Каждый из них в коде живёт десятками копий, и каждый раз кто-то новый их пишет руками.

Дальше — по одному примеру на шаблон. Где он, чем плох, и что с ним можно сделать.

---

## Шаблон 1. «Загрузить или 404»

### Где встречается

Везде. Реально — в каждом *Edit*/*Delete*-хэндлере по всем модулям: *Discounts*, *Segments*, *Certificates*, *Promocodes*, *Promotions*. Скелет один:

```csharp
// Discounts/RequestHandler.cs — Edit
public async ValueTask<DiscountDto> Handle(EditDiscountRequest request, CancellationToken ct)
{
    var corporationId = await _accountService.GetCorporation();
    var discount = await _repository.Get(corporationId, request.Id);
    if (discount == null)
        throw new ApiException(Enums.ApiStatus.ERR_NOT_FOUND_ITEM, "Скидка не найдена");

    _service.Update(discount, request.Name, ...);
    await _repository.Save(discount);
    return DiscountDto.Map(discount);
}

// Discounts/RequestHandler.cs — Delete (тот же скелет)
public async ValueTask<Unit> Handle(DeleteDiscountRequest request, CancellationToken ct)
{
    var corporationId = await _accountService.GetCorporation();
    var discount = await _repository.Get(corporationId, request.Id);
    if (discount == null)
        throw new ApiException(Enums.ApiStatus.ERR_NOT_FOUND_ITEM, "Скидка не найдена");

    discount.Delete();
    await _repository.Save(discount);
    return await Unit.ValueTask;
}

// Segments — то же самое, отличается одно слово
var segment = await _repository.Get(corporationId, request.Id);
if (segment == null)
    throw new ApiException(Enums.ApiStatus.ERR_NOT_FOUND_ITEM, "Сегмент не найден");

// Certificates — снова те же три строки
var certificate = await _repository.Get(corporationId, request.Id);
if (certificate == null)
    throw new ApiException(Enums.ApiStatus.ERR_NOT_FOUND_ITEM, "Сертификат не найден");
```

Если посчитать по проекту — это минимум 12–15 одинаковых блоков по 3 строки, и каждое сообщение об ошибке написано руками заново. И стиль этих сообщений тоже плавает: где-то «не найдена», где-то «не найден», где-то по правилам, где-то нет.

### Почему это раздражает

Если присмотреться, в этих трёх строках смешаны две разные вещи: спросить у репозитория и убедиться, что что-то нашли. Они всегда идут вместе, но синтаксически разнесены. И каждый, кто пишет новый Edit-хэндлер, обязан копировать обе — иначе либо вылетит **NullReferenceException**, либо API вернёт что-то невразумительное вместо 404.

### Абстракция: extension **GetOrThrow**

Достаточно одного маленького метода:

```csharp
public static class RepositoryExtensions
{
    public static async Task<T> GetOrThrow<T>(this Task<T?> getTask, string entityName = null)
        where T : class
    {
        var entity = await getTask;
        if (entity == null)
            throw new ApiException(
                Enums.ApiStatus.ERR_NOT_FOUND_ITEM,
                $"{entityName ?? typeof(T).Name} не найден(а)");
        return entity;
    }
}
```

Если захочется красивых русских названий — можно завести словарик **Dictionary<Type, string>** где-то рядом, и вызывать без второго параметра. Но это уже косметика, основное — что место принятия решения «как мы говорим про 404» теперь одно.

### Стало

```csharp
// Edit
public async ValueTask<DiscountDto> Handle(EditDiscountRequest request, CancellationToken ct)
{
    var corporationId = await _accountService.GetCorporation();
    var discount = await _repository.Get(corporationId, request.Id).GetOrThrow("Скидка");

    _service.Update(discount, request.Name, ...);
    await _repository.Save(discount);
    return DiscountDto.Map(discount);
}

// Delete
public async ValueTask<Unit> Handle(DeleteDiscountRequest request, CancellationToken ct)
{
    var corporationId = await _accountService.GetCorporation();
    var discount = await _repository.Get(corporationId, request.Id).GetOrThrow("Скидка");

    discount.Delete();
    await _repository.Save(discount);
    return await Unit.ValueTask;
}
```

### Что улучшили

Три строки превратились в одну. По проекту — это минус 30–40 строк одинакового шума. Но главное даже не в этом. Главное в том, что если завтра захочется, например, логировать каждый 404 или возвращать структурированный ответ с подсказками — я правлю в одном месте, а не в пятнадцати. И никто не сможет завести новый хэндлер, в котором «забыли проверить на null» — потому что вызов **Get** без **GetOrThrow** сразу выглядит подозрительно, на него спотыкаешься глазами.

---

## Шаблон 2. «try-Commit-catch-Rollback-finally-Dispose» вокруг UoW

### Где встречается

В *Calculations/CheckIn/RequestHandler* и других местах, где надо что-то записать в базу под транзакцией:

```csharp
public async Task<(bool, string)> Handle(CheckInGuestRequest request, CancellationToken ct)
{
    Calculation calculation;
    var repository = _unitOfWork.CalculationRepository;
    var antifraudRepository = _unitOfWork.AntifraudRepository;

    try
    {
        calculation = await _factory.GetAndCheckin(repository, antifraudRepository,
                                                    request.Token, request.CorporationId, request.GuestId);
        await repository.Save(calculation);
        _unitOfWork.Commit();
    }
    catch (Exception ex)
    {
        _unitOfWork.Rollback();
        _logger.LogError(ex, ex.Message);
        return (false, null);
    }
    finally
    {
        _unitOfWork.Dispose();
    }

    return (true, calculation.OrderNumber);
}
```

И вот это — буквально тот самый пример из материала, только в .NET-обёртке. Помните **getRoutingTable() / releaseRoutingTable()** с **try-finally**? Это оно. Открыли ресурс, поработали, закрыли — и всё это вокруг трёх строк полезного дела.

### Почему это раздражает

Каждый, кто пишет такой хэндлер, должен помнить целый список правил: Commit только в успехе, Rollback только в catch, Dispose обязательно в finally, finally не должно есть исключение, а ещё неплохо бы залогировать. Семь правил вокруг трёх строк работы. И ничего из этого не выражено в типах — это просто конвенция, которую кто-то напишет правильно, а кто-то нет. Особенно если торопится.

Хеттингер в материале как раз и говорит про этот случай: сама конструкция **try-finally** вокруг ресурса — она и есть шаблон. Не отдельные строчки, а вся обвязка целиком. И в Python для неё придумали **with**. В C# своего **with** нет, но есть штука, которая работает точно так же — лямбда, переданная в метод.

### Абстракция: **IUnitOfWork.Execute**

Вся обвязка переезжает внутрь самого UoW, а наружу торчит только «дай мне функцию, которая хочет поработать»:

```csharp
public interface IUnitOfWork : IDisposable
{
    ICalculationRepository CalculationRepository { get; }
    IBonusRepository BonusRepository { get; }
    IAntifraudRepository AntifraudRepository { get; }

    Task<T> Execute<T>(Func<IUnitOfWork, Task<T>> work);
    Task     Execute(Func<IUnitOfWork, Task> work);
}

public sealed class UnitOfWork : IUnitOfWork
{
    // ... поля и репозитории как раньше

    public async Task<T> Execute<T>(Func<IUnitOfWork, Task<T>> work)
    {
        try
        {
            var result = await work(this);
            Commit();
            return result;
        }
        catch
        {
            Rollback();
            throw;
        }
        finally
        {
            Dispose();
        }
    }

    public async Task Execute(Func<IUnitOfWork, Task> work)
        => await Execute<object>(async uow => { await work(uow); return null; });
}
```

### Стало

```csharp
public Task<(bool, string)> Handle(CheckInGuestRequest request, CancellationToken ct)
{
    return _unitOfWork.Execute(async uow =>
    {
        var calculation = await _factory.GetAndCheckin(
            uow.CalculationRepository, uow.AntifraudRepository,
            request.Token, request.CorporationId, request.GuestId);

        await uow.CalculationRepository.Save(calculation);

        return (true, calculation.OrderNumber);
    });
    // Если внутри что-то упало — Rollback и Dispose уже произошли,
    // исключение полетит наружу, и его поймает общий middleware.
    // Не надо ничего мапить в (false, null) руками.
}
```

### Что улучшили

13 строк обвязки уехали в одну строчку **Execute(...)**. И — это даже важнее — забыть что-либо теперь нельзя физически. Commit ставится только после успешной работы. Rollback ставится только в catch. Dispose — всегда. Вызвать Commit дважды или забыть его — нет такого способа. Порядок вшит в саму абстракцию, и у разработчика просто не остаётся возможности сделать неправильно.

Ну и приятный побочный эффект — **_logger.LogError** и преобразование исключения в **(false, null)** тоже ушли. Это не задача хэндлера, это задача middleware на уровне веб-приложения. Хэндлер теперь делает то, что должен делать хэндлер: что-то полезное, и всё.

---

## Шаблон 3. «try-publish-success / catch-log-publish-failure» в обработчиках саги

### Где встречается

В *SagaImportConsumer* — четыре метода **Consume**, и все четыре — одно и то же:

```csharp
public async Task Consume(ConsumeContext<IImportRestaurantLicenses> context)
{
    try
    {
        await _mediator.Send(new ImportRestaurantsRequest(context.Message.CorporationId));
        await context.Publish<ISuccessfulRestaurantLicensesImported>(new {
            context.Message.RequestId, context.Message.CorporationId });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, $"Log Error IFailedRestaurantLicensesImported {ex.Message}");
        await context.Publish<IFailedRestaurantLicensesImported>(new {
            context.Message.RequestId, context.Message.CorporationId,
            ErrorDecription = ex.Message });
    }
}

public async Task Consume(ConsumeContext<ICreateRestaurants> context)
{
    try
    {
        await context.Publish<ISuccessfulRestaurantsCreated>(new {
            context.Message.RequestId, context.Message.CorporationId });
    }
    catch (Exception ex)
    {
        await context.Publish<IFailedRestaurantsCreated>(new {
            context.Message.RequestId, context.Message.CorporationId,
            ErrorDecription = ex.Message });
    }
}

// ISetCorporationSettings, ICompleteImportCorporation — те же 12 строк, другие типы
```

Четыре копии в одном файле. И таких консьюмеров в системе не один.

### Почему это раздражает

Бизнес-логика в этих 12 строках — это содержимое try-блока, и часто это одна строка. Всё остальное — обвязка «попробуй, сообщи об успехе, поймай, залогируй, сообщи о провале». Эта обвязка живёт во всех четырёх методах, и в каждом её надо написать заново.

И ещё одна вещь, которая лично меня цепляет. Имена событий *Successful\<X\>* и *IFailed\<X\>* — это явная пара. Они появляются и используются вместе. Но нигде в коде эта пара не выражена — её надо просто *знать*. Один Consume забыл logger.LogError, другой забыл — и это не баг, это просто «у каждого метода своя жизнь». А должно быть наоборот: парность вшита, и нельзя добавить успех без провала.

### Абстракция: типизированная пара событий + общий обработчик

Парность выражаем явно — **SagaStepEvents<TInput>**, в котором лежат оба типа событий сразу. А цикл «try-publish-catch-log-publish» уносим в extension:

```csharp
// Описание пары событий — связаны навсегда
public record SagaStepEvents<TInput>(
    Func<TInput, object> Success,
    Func<TInput, string, object> Failure)
    where TInput : class
{
    public Type SuccessType { get; init; }
    public Type FailureType { get; init; }
}

// Обвязка — одна на все шаги саги
public static class SagaStepExtensions
{
    public static async Task ExecuteStep<TMessage>(
        this ConsumeContext<TMessage> context,
        SagaStepEvents<TMessage> events,
        ILogger logger,
        Func<TMessage, Task> work = null)
        where TMessage : class
    {
        try
        {
            if (work != null)
                await work(context.Message);

            await context.Publish(events.SuccessType, events.Success(context.Message));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Saga step {Step} failed: {Message}",
                            typeof(TMessage).Name, ex.Message);
            await context.Publish(events.FailureType, events.Failure(context.Message, ex.Message));
        }
    }
}

// Шаги саги — таблица, которую видно одним взглядом
internal static class ImportSagaSteps
{
    public static readonly SagaStepEvents<IImportRestaurantLicenses> RestaurantLicenses =
        new(
            m => new { m.RequestId, m.CorporationId },
            (m, err) => new { m.RequestId, m.CorporationId, ErrorDecription = err })
        {
            SuccessType = typeof(ISuccessfulRestaurantLicensesImported),
            FailureType = typeof(IFailedRestaurantLicensesImported),
        };

    public static readonly SagaStepEvents<ICreateRestaurants> CreateRestaurants =
        new(/* ... та же структура для своей пары событий ... */);

    // ISetCorporationSettings, ICompleteImportCorporation
}
```

### Стало

```csharp
public class SagaImportConsumer :
    IConsumer<IImportRestaurantLicenses>,
    IConsumer<ICreateRestaurants>,
    IConsumer<ISetCorporationSettings>,
    IConsumer<ICompleteImportCorporation>
{
    private readonly IMediator _mediator;
    private readonly ILogger<SagaImportConsumer> _logger;

    public SagaImportConsumer(IMediator mediator, ILogger<SagaImportConsumer> logger)
    {
        _mediator = mediator;
        _logger = logger;
    }

    public Task Consume(ConsumeContext<IImportRestaurantLicenses> ctx) =>
        ctx.ExecuteStep(ImportSagaSteps.RestaurantLicenses, _logger,
            m => _mediator.Send(new ImportRestaurantsRequest(m.CorporationId)).AsTask());

    public Task Consume(ConsumeContext<ICreateRestaurants> ctx) =>
        ctx.ExecuteStep(ImportSagaSteps.CreateRestaurants, _logger);

    public Task Consume(ConsumeContext<ISetCorporationSettings> ctx) =>
        ctx.ExecuteStep(ImportSagaSteps.CorporationSettings, _logger);

    public Task Consume(ConsumeContext<ICompleteImportCorporation> ctx) =>
        ctx.ExecuteStep(ImportSagaSteps.CompleteImport, _logger,
            m => _mediator.Send(new CompleteImportCorporationRequest(m.CorporationId)).AsTask());
}
```

### Что улучшили

Каждый **Consume** теперь — одна строка. Обвязка «try-catch-log-publish» переехала в один метод на всех. Парность Success/Fail событий теперь видна прямо в коде: чтобы создать **SagaStepEvents**, нужно указать оба типа — иначе не скомпилируется. Логирование везде одинаковое, причём с осмысленным именем шага, а не «Log Error IFailedRestaurantLicensesImported» (это, кстати, опечатка из исходника, тоже жизнь).

Самое приятное: добавить пятый шаг — это одна запись в **ImportSagaSteps** и одна строка в консьюмере. Не «скопируй 12 строк, не забудь поменять все типы, не забудь логирование, не забудь точно вписать те же поля».

---

## Что в итоге

Три шаблона разной природы. Первый — линейный, «получи и проверь». Второй — про ресурс с открытием и закрытием. Третий — обвязка с парными результатами. Решения тоже разные: где-то хватает extension-метода на пять строк, где-то нужна функция высшего порядка с лямбдой, где-то — отдельная маленькая модель.

Но общая идея у всех трёх одна. Это не дублирование кода в смысле «копипасты строк» — конкретные строки иногда даже разные. Это дублирование *самого узора* того, как код устроен. «Получи и проверь», «открой-поработай-закрой», «попробуй и доложи результат» — каждый раз руками, в каждом новом файле.

В материале есть хорошая мысль на эту тему: вынести в функцию повторяющееся выражение — это видят все. А вот вынести повторяющуюся управляющую конструкцию через лямбду или контекст-менеджер — этого почему-то почти никто не делает, хотя именно тут обычно прячется самое большое сокращение кода.

Мои три случая — как раз про второй вариант. И сокращение получилось серьёзное, но дело даже не в количестве строк. Дело в том, что после такой абстракции типичные ошибки становятся буквально невозможными — забыть проверку на null, забыть Rollback, забыть failure-событие. Не «не положено», а именно невозможно. И в этом, наверное, главный смысл всей этой возни с правильным дизайном.