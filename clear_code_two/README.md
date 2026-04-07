# Отчёт: Плохой стиль на уровне реализации

## Введение

Взял пять категорий проблем и прошёлся по живому коду проекта. Первая реакция на некоторые места была «ну, так и задумано» — но чем дольше смотришь, тем яснее видишь, где намерение потерялось, а где просто никто не остановился и не переспросил себя: «а зачем это вообще публичное?»

Главное, что вынес: большинство проблем на этом уровне не про синтаксис и не про архитектуру. Они про коммуникацию. Семь параметров в конструкторе — это не ошибка, это отсутствие диалога между автором и читателем кода. Два независимых *if* вместо *if/else if* — это не баг, это потерянное намерение.

---

## 1.1. Методы, которые используются только в тестах

Честно говоря, когда первый раз увидел *InitDb* в базовом классе *Condition*, подумал: «ну, это же просто инициализация, норм». Но потом посмотрел, откуда он вызывается — и оказалось, что в продакшен-коде его вызывает только *ConditionFactory*, а в тестах — каждый тест-класс вручную через хелперы *CreateCondition*. То есть метод **internal**, хотя его единственная цель — пробросить мок в тест. Это дыра в инкапсуляции, замаскированная под «удобство».

### Исходный код

```csharp
// Condition.cs — базовый класс
public abstract class Condition
{
    // Метод internal — виден во всей сборке.
    // В тестах вызывается напрямую, чтобы подменить контекст.
    // Проблема: существование этого метода продиктовано тестами, а не доменом.
    internal void InitDb(IConditionContext context)
    {
        ConditionContext = context;
    }

    protected IConditionContext ConditionContext { get; private set; }
}

// GuestSegmentsTests.cs — тест
private GuestSegments CreateCondition(List<Guid> segmentIds)
{
    var condition = new GuestSegments { SegmentIds = segmentIds };
    condition.InitDb(_mockConditionContext.Object);   // ← вот зачем метод существует
    condition.InitBenefit(_corporationId, _benefitId, BenefitType.Promotion);
    return condition;
}
```

### Исправленный код

```csharp
// Condition.cs — контекст через конструктор, InitDb исчезает как явление
public abstract class Condition
{
    // Зависимость явная и обязательная.
    // Компилятор не даст создать условие без контекста — ни в продакшене, ни в тесте.
    protected Condition(IConditionContext context)
    {
        ConditionContext = context;
    }

    protected IConditionContext ConditionContext { get; }
}

// GuestSegmentsTests.cs — тест теперь передаёт мок через конструктор, как и должен
private GuestSegments CreateCondition(List<Guid> segmentIds)
{
    return new GuestSegments(_mockConditionContext.Object)
    {
        SegmentIds = segmentIds
    };
}
```

### Что изменилось

*InitDb* не нужен вообще. Тест передаёт мок туда же, куда продакшен передаёт реальный контекст — в конструктор. Намерение теперь видно из сигнатуры: у условия всегда есть контекст, это не опциональное состояние.

Ещё один момент, который поначалу не очевиден: пока *InitDb* существует, *ConditionContext* технически может быть *null* — никто не гарантирует, что метод вызвали до *IsMatch*. С конструктором этой проблемы нет.

---

## 1.2. Цепочки методов

Внутри *CalculationFactory.CreateByPhone* и *CreateByToken* — пять последовательных вызовов, каждый зависит от предыдущего: получить контекст → создать гостя → проверить антифрод → проверить исключения → сконструировать расчёт. Всё это в одном методе, без имён для промежуточных концепций. Читаешь и видишь набор строк, а не намерение.

Особенно показательно то, что *CreateByToken* повторяет шаги 2–4 один в один — только перед ними стоит декодирование токена. То есть дублирование цепочки живёт прямо в соседнем методе.

### Исходный код

```csharp
public async Task<Calculation> CreateByPhone(
    ICalculationRepository repository,
    IAntifraudRepository antifraudRepository,
    long restaurantId, string guestPhone, Guid? idempotencyKey = null)
{
    // Шаг 1
    var context = await repository.GetContext(restaurantId);

    // Шаг 2 — зависит от шага 1
    var guest = await _guestFactory.GetOrCreate(context.CorporationId, guestPhone);

    // Шаг 3 — зависит от шага 2
    var antifraudResult = await _antifraudService.CheckAllow(
        antifraudRepository, context.CorporationId, restaurantId, guest.Id);

    // Шаг 4 — снова зависит от шага 2
    var guestExcluded = await _bonusExclusionsRepository.IsExcluded(
        context.CorporationId, guest.Id);

    // Шаг 5 — зависит от всего
    return new Calculation(context, guest, guestExcluded, antifraudResult, idempotencyKey);
}
```

### Исправленный код

```csharp
public async Task<Calculation> CreateByPhone(
    ICalculationRepository repository,
    IAntifraudRepository antifraudRepository,
    long restaurantId, string guestPhone, Guid? idempotencyKey = null)
{
    var context = await repository.GetContext(restaurantId);
    // Шаги 2–4 теперь живут под понятным именем.
    // CreateByToken использует тот же метод — дублирования нет.
    var guestContext = await ResolveGuest(antifraudRepository, context, restaurantId, guestPhone);

    return new Calculation(context, guestContext.Guest, guestContext.IsExcluded,
        guestContext.AntifraudResult, idempotencyKey);
}

// Метод с именем — «разреши гостя со всеми его проверками».
// Это самостоятельная операция, а не просто набор строк.
private async Task<GuestContext> ResolveGuest(
    IAntifraudRepository antifraudRepository,
    CalculationContext context,
    long restaurantId, string guestPhone)
{
    var guest = await _guestFactory.GetOrCreate(context.CorporationId, guestPhone);
    var antifraudResult = await _antifraudService.CheckAllow(
        antifraudRepository, context.CorporationId, restaurantId, guest.Id);
    var isExcluded = await _bonusExclusionsRepository.IsExcluded(context.CorporationId, guest.Id);

    return new GuestContext(guest, antifraudResult, isExcluded);
}

// Локальный record — не DTO, просто контейнер для трёх связанных результатов
private record GuestContext(Guest Guest, AntifraudResult AntifraudResult, bool IsExcluded);
```

### Что изменилось

Пять строк без имён превратились в два шага с именами. *ResolveGuest* теперь переиспользуется в *CreateByToken* — там после декодирования токена идут ровно те же шаги. Тестировать тоже проще: *ResolveGuest* можно проверить отдельно, не поднимая весь сценарий создания расчёта.

---

## 1.3. Слишком большой список параметров

*CreateCalculationRequest* — семь параметров в конструкторе. Последний — *IEnumerable* кортежа с семью вложенными полями. Это нечитаемо уже при написании, а при ревью — просто боль: непонятно, что передаётся на каком месте, IDE не подскажет имя поля, а сам тип нигде не переиспользуется, хотя блюда явно нужны и в других запросах.

### Исходный код

```csharp
public class CreateCalculationRequest : IRequest<CalculationDto>
{
    public CreateCalculationRequest(
        long restaurantId,
        string token,
        string phone,
        Guid orderId,
        string orderNumber,
        // Анонимный кортеж с 7 полями — IDE не подскажет, что куда
        IEnumerable<(long Id, string Name, decimal Quantity, decimal Price,
                     decimal Amount, decimal? MinPrice, decimal? MaxPrice)> dishes)
    {
        RestaurantId = restaurantId;
        Token = token;
        Phone = phone;
        OrderId = orderId;
        OrderNumber = orderNumber;
        Dishes = dishes;
    }
}
```

### Исправленный код

```csharp
// Явный DTO вместо кортежа.
// Называется так же, как в других запросах — CalcDishDto уже существует в проекте,
// здесь просто следуем той же конвенции.
public class CreateDishDto
{
    public long Id { get; init; }
    public string Name { get; init; }
    public decimal Quantity { get; init; }
    public decimal Price { get; init; }
    public decimal Amount { get; init; }
    public decimal? MinPrice { get; init; }
    public decimal? MaxPrice { get; init; }
}

public class CreateCalculationRequest : IRequest<CalculationDto>
{
    public CreateCalculationRequest(
        long restaurantId,
        string token,
        string phone,
        Guid orderId,
        string orderNumber,
        IEnumerable<CreateDishDto> dishes)  // ← читается, переиспользуется, документируется
    {
        // ...
    }
}
```

### Что изменилось

Анонимный кортеж ушёл. *CreateDishDto* теперь можно задокументировать, добавить валидацию, переиспользовать в *EditCalculationRequest* — там та же структура блюда. Следующий шаг, если нужен — объединить *token* и *phone* в *GuestIdentifier*, и конструктор сожмётся до пяти параметров с понятными концепциями.

---

## 1.4. Странные решения

В *PreCalculations.RequestHandler* и *Calculations.RequestHandler* гость идентифицируется двумя способами — по токену или по телефону. Это взаимоисключающие варианты, но реализованы они как два **независимых** *if*. Если передать оба — выполнятся оба блока, второй молча перезапишет первый. Намерения не видно: это приоритет телефона над токеном? Или просто никто не подумал?

### Исходный код

```csharp
Calculation calculation = null;

// Если по токену
if (!string.IsNullOrEmpty(request.Token))
{
    calculation = await _factory.CreateByToken(repository, antifraudRepository,
        request.RestaurantId, request.Token);
}

// Если по телефону — независимый if, не else if.
// При наличии обоих значений этот блок выполнится вторым и перезапишет первый.
if (!string.IsNullOrEmpty(request.Phone))
{
    calculation = await _factory.CreateByPhone(repository, antifraudRepository,
        request.RestaurantId, request.Phone);
}

if (calculation == null)
{
    throw new ApiException(Enums.ApiStatus.FORBIDDEN_WITH_MESSAGE, "Антифрод: превышен лимит заказов");
}
```

### Исправленный код

```csharp
Calculation calculation;

if (!string.IsNullOrEmpty(request.Token))
{
    // Токен имеет приоритет — это теперь читается явно
    calculation = await _factory.CreateByToken(repository, antifraudRepository,
        request.RestaurantId, request.Token);
}
else if (!string.IsNullOrEmpty(request.Phone))
{
    calculation = await _factory.CreateByPhone(repository, antifraudRepository,
        request.RestaurantId, request.Phone);
}
else
{
    // Ошибка стала честной: проблема не в антифроде, а в том, что вызывающая
    // сторона не передала ни один из способов идентификации гостя
    throw new ApiException(Enums.ApiStatus.ERR_PARSE_REQUEST,
        "Необходимо указать токен гостя или номер телефона");
}
```

### Что изменилось

*if/else if/else* вместо двух независимых *if*. Теперь приоритет токена над телефоном явный. Ошибка в *else* стала точной — раньше сообщение «Антифрод: превышен лимит заказов» вводило в заблуждение: антифрод здесь ни при чём, просто не передали идентификатор гостя.

Именно такие «странные решения» — самые дорогие при отладке: код работает в 99% случаев, а падает в одном специфическом сценарии, который сложно воспроизвести.

---

## 1.5. Чрезмерный результат

В *Closes.RequestHandler* есть ранний возврат: если чеки не изменились (*hasChange == false*), метод возвращает расчёт как есть. Клиенту в этом случае нужны буквально два поля — *Id* и *Status*. Вместо этого маппится полный *CloseCalculationDto* со всеми восемью полями, включая *TotalDiscount*, *TotalAmount*, *TotalAmountWithDiscount*. Это лишняя работа — и сигнал для читателя, что оба пути возврата семантически одинаковы, хотя на деле один из них значит «ничего не изменилось».

### Исходный код

```csharp
var hasChange = _service.Close(calculation, request.Receipts).ThrowIfError();

if (!hasChange)
    // Маппим все 8 полей, хотя нужны только Id и Status.
    // Читатель видит MapFrom — и думает, что оба возврата равнозначны.
    return CloseCalculationDto.MapFrom(calculation);

// ... дальше реальная работа: бонусы, история, события
await bonusRepository.CloseHold(calculation);
await bonusRepository.AwardBonuses(calculation);
await repository.Save(calculation);
// ...
return CloseCalculationDto.MapFrom(calculation);
```

*CloseCalculationDto* содержит 8 полей — а при *hasChange == false* нужны только 2.

### Исправленный код

```csharp
var hasChange = _service.Close(calculation, request.Receipts).ThrowIfError();

if (!hasChange)
    // Явно: возвращаем только то, что нужно при отсутствии изменений.
    // Читатель видит разницу — это не полный результат, а статусный ответ.
    return CloseCalculationDto.MapNoChanges(calculation);

// ... дальше реальная работа
await bonusRepository.CloseHold(calculation);
await bonusRepository.AwardBonuses(calculation);
await repository.Save(calculation);
// ...
return CloseCalculationDto.MapFrom(calculation);

// Добавить в CloseCalculationDto:
public static CloseCalculationDto MapNoChanges(Domain.Calculations.Calculation calculation)
{
    return new CloseCalculationDto
    {
        Id = calculation.Id,
        Status = calculation.Status.ToString(),
        Version = calculation.Version,
        UpdatedAt = calculation.UpdateDate ?? calculation.CreateDate
        // Остальные поля — default. Клиенту они не нужны.
    };
}
```

### Что изменилось

Два пути возврата теперь семантически различимы — *MapFrom* значит «полный результат», *MapNoChanges* значит «расчёт не изменился, вот статус». Если в будущем *MapFrom* начнёт тащить коллекции блюд или историю — ранний возврат не замедлится вместе с ним.

---

## Общая рефлексия

Честно говоря, в начале я думал, что уровень реализации — это «для разминки», всё очевидно. Оказалось, что нет. Самое интересное — это не *if* вместо *if/else if* и не кортеж вместо DTO. Самое интересное — это то, что каждая из этих проблем прячет потерянное намерение.

*InitDb* существует потому, что кто-то не захотел менять конструктор, а тесты надо было написать. Два независимых *if* появились потому, что второй кейс добавили позже и не переспросили себя: «а что если оба параметра переданы?» Кортеж из семи полей — потому что казалось быстрее, чем создавать отдельный класс.

Для себя сформулировал простой вопрос, который помогает: «если я уберу этот метод/параметр/тип — что сломается и почему?» Если ответ «только тест» — значит, метод существует ради теста, а не ради домена. Если ответ «ничего конкретного, просто неудобно» — значит, скорее всего, это чрезмерный результат или лишний параметр. Если ответ непонятен — значит, намерение потерялось, и его надо восстановить явно в коде, а не в голове.