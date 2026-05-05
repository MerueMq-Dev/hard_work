# Отчёт: Три правила простого проектного дизайна

## Введение

Когда читаешь про правила простого дизайна — кажется, что всё очевидно. Не передавай примитивы там, где нужны типы. Не делай дефолтных конструкторов. Не оставляй способа вызвать методы в неправильном порядке. Но одно дело читать про шахматный пример, другое — находить эти вещи в собственном коде. Я прошёлся по проекту и нашёл шесть живых примеров — по два на каждое правило.

Самое интересное, что все шесть я поначалу не воспринимал как проблемы. Каждый из них выглядел как «ну, осторожный разработчик разберётся». Именно это и описывает Норман: когда система плохо спроектирована, люди винят себя, а не систему. Мой подбор намеренно делает акцент на **первом** правиле — оно самое ёмкое и встречается в проекте чаще остальных.

---

## Правило 1. Запрещать ошибочное поведение на уровне интерфейса

Суть правила: если некоторая последовательность вызовов методов приводит к ошибке или недопустимому состоянию — это не проблема документации. Это проблема интерфейса класса. Правильный дизайн делает ошибочный вызов физически невозможным.

### Пример 1.1. *Calculation* — Close без Guest

В *Calculation* есть статусная модель. Статус *created* означает «заявка без Гостя — нельзя вызывать */close*». Это написано прямо в комментарии к перечислению:

```csharp
public enum CalculationStatus
{
    created, // без Гостя - нельзя вызывать /close
    active,  // с Гостем
    closed,
    refunded,
    canceled,
}
```

Сейчас защита есть, но живёт в *CalculationService.Close* в виде двух последовательных проверок:

```csharp
// CalculationService.cs — было
if (calculation.Status.IsCompleted())
    return CalculationErrors.CloseNotAllowed(calculation.Status);

if (calculation.Status == CalculationStatus.created)
    return CalculationErrors.CloseNotAllowed(calculation.Status);
```

Это уже лучше, чем ничего — ошибка хотя бы явная. Но проблема в том, что запрет живёт в *сервисе*, а не в *классе*. Другой сервис, другой хэндлер, прямой вызов `calculation.Close()` в тесте — и запрет обходится незаметно.

Корень проблемы глубже: *Guest* устанавливается через публичный сеттер отдельно от создания объекта. Расчёт можно создать и тут же попробовать закрыть, минуя сервис. Дизайн класса позволяет это сделать.

```csharp
// Было: Guest — публичный сеттер, Close — публичный метод
// Снаружи ничто не мешает:
var calc = new Calculation(...);
calc.Close(); // статус created, Guest == null — но метод вызвался
```

```csharp
// Стало: разделение на два явных способа создания.
// Расчёт без гостя и расчёт с гостем — это разные состояния, и они выражены в API класса.
public class Calculation
{
    private Calculation(AggregateContext context, Guid orderId, ...)
    {
        Status = CalculationStatus.created;
        // Guest == null — это pending-расчёт, Close недоступен
    }

    private Calculation(AggregateContext context, Guest guest, Guid orderId, ...)
    {
        Guest = guest ?? throw new ArgumentNullException(nameof(guest));
        Status = CalculationStatus.active;
        // Только этот конструктор создаёт расчёт, к которому можно вызвать Close
    }

    // Явные фабричные методы вместо «угадай, какой конструктор тебе нужен»
    public static Calculation CreatePending(AggregateContext context, Guid orderId, ...)
        => new Calculation(context, orderId, ...);

    public static Calculation CreateWithGuest(AggregateContext context, Guest guest, Guid orderId, ...)
        => new Calculation(context, guest, orderId, ...);

    // Close теперь проверяет «верный ли статус» — а статус active гарантирован конструктором.
    // Внешняя проверка в сервисе становится не нужна.
    internal ErrorOr<bool> Close(...)
    {
        if (Status != CalculationStatus.active)
            return CalculationErrors.CloseNotAllowed(Status);
        // ...
    }
}
```

**Что улучшили.** Запрет на *Close* без гостя перенесён с уровня сервиса на уровень конструктора. Теперь *active*-расчёт нельзя создать без гостя — это физически невозможно. Разработчик не может вызвать *Close* у *created*-расчёта по невнимательности, потому что такой объект просто не имеет шанса оказаться в недопустимом состоянии. Защита переехала из «правильно вызови сервис» в «правильно сконструируй объект».

---

### Пример 1.2. *Refund* — возврат без проверки внутри *Calculation*

В *CalculationService.Refund* — цепочка из трёх условий:

```csharp
// Было: три отдельных условия в сервисе
if (calculation.Status == CalculationStatus.canceled
    || calculation.Status == CalculationStatus.active
    || calculation.Status == CalculationStatus.created)
    return CalculationErrors.RefundNotAllowed(calculation.Status);
```

Это читается как «возврат разрешён только из *closed* или *refunded*». Но это знание живёт в сервисе, а не в самом расчёте. Другой разработчик, вызывая *calculation.AddToRefund* напрямую (например, в тесте), не получит никакой защиты.

Хуже того: внутри *Calculation.AddToRefund* статус меняется без проверки, можно ли вообще это делать в текущем состоянии. Условие там есть только на «были ли вообще изменения». Само присваивание `Status = CalculationStatus.refunded` дублируется — оно стоит и в цикле обхода блюд, и в приватном методе *Refunded()*. Это ещё один симптом: код «лечит» отсутствие инкапсуляции через дублирование.

```csharp
// Было: внутри Calculation.AddToRefund — нет проверки статуса,
// и при этом смена статуса прописана дважды
internal bool AddToRefund(IEnumerable<...> receipts, IEnumerable<...> dishesToRefund)
{
    bool hasChange = false;

    foreach (var r in receipts) { /* ... */ hasChange = true; }

    if (hasChange)
    {
        foreach (var d in dishesToRefund)
        {
            // ... добавляем блюдо
            Status = CalculationStatus.refunded;   // (1) — здесь
        }
        Refunded();                                 // (2) — и внутри тоже
    }
    return hasChange;
}

private void Refunded()
{
    Status = CalculationStatus.refunded;            // дубль (2)
    UpdateDate = DateTimeOffset.UtcNow;
    ConcurrencyToken = Guid.NewGuid();
    UpVersion();
}
```

```csharp
// Стало: расчёт сам охраняет переход в refunded.
// Дублирование смены статуса убрано — единственная точка изменения внутри Refunded().
internal ErrorOr<bool> AddToRefund(IEnumerable<...> receipts, IEnumerable<...> dishesToRefund)
{
    // Защита живёт внутри расчёта — нельзя возвращать заявку из неправильного статуса,
    // независимо от того, кто вызывает: сервис, тест, новый хэндлер
    if (Status != CalculationStatus.closed && Status != CalculationStatus.refunded)
        return CalculationErrors.RefundNotAllowed(Status);

    bool hasChange = false;
    foreach (var r in receipts) { /* ... */ hasChange = true; }

    if (hasChange)
    {
        foreach (var d in dishesToRefund) { /* добавляем блюдо */ }
        Refunded(); // единственная точка смены статуса
    }
    return hasChange;
}
```

**Что улучшили.** Проверка переехала из сервиса в сам объект. Теперь *Calculation* сам охраняет своё состояние — неважно, откуда вызван *AddToRefund*: из сервиса, из теста или из нового хэндлера. Нарушить переход статуса физически невозможно. Заодно убрано дублирование: смена статуса теперь только в *Refunded()* — изменить логику этого перехода нужно в одном месте, а не в двух.

---

## Правило 2. Отказаться от дефолтных конструкторов без параметров

Суть правила: если объект не имеет смысла без каких-то данных — не давай его создать без этих данных. Дефолтный конструктор приглашает к созданию «пустых» объектов, которые потом взрываются в рантайме.

### Пример 2.1. *ScheduledJob* — объект без обязательных полей

*ScheduledJob* — класс для хранения отложенных задач. Открываешь и видишь:

```csharp
// Было: все поля с default! — компилятор доволен, рантайм — нет
public class ScheduledJob
{
    public string Id { get; set; } = default!;
    public string JobType { get; set; } = default!;
    public DateTimeOffset ExecutionDate { get; set; }
    public string Status { get; set; } = default!;     // ← плюс ещё и строка вместо enum
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? CompletedAt { get; set; }
    public string? Error { get; set; }
}
```

*default!* — это «я обещаю, что здесь будет значение, доверяй мне». Компилятор отключает проверку на *null*, и любой код может создать *ScheduledJob* без *Id*, *JobType* и *Status*. Попытка сохранить такой объект в БД — ошибка в рантайме.

Этот класс — учебный пример сразу двух правил. Помимо отсутствия конструктора, *Status* ещё и хранится как `string` (правило 3). Если завтра кто-то напишет `job.Status = "complete"` (а не `"completed"`) — компилятор пропустит, тесты могут не поймать, а в БД запишется значение, которое никакой код не ожидает.

```csharp
// Стало: конструктор требует всё обязательное сразу + Status становится enum
public class ScheduledJob
{
    public ScheduledJob(string id, string jobType, DateTimeOffset executionDate)
    {
        Id = id ?? throw new ArgumentNullException(nameof(id));
        JobType = jobType ?? throw new ArgumentNullException(nameof(jobType));
        ExecutionDate = executionDate;
        Status = JobStatus.Pending; // начальный статус известен — задаётся сразу
        CreatedAt = DateTimeOffset.UtcNow;
    }

    public string Id { get; private set; }
    public string JobType { get; private set; }
    public DateTimeOffset ExecutionDate { get; private set; }
    public JobStatus Status { get; private set; }    // enum, а не string
    public DateTimeOffset CreatedAt { get; private set; }
    public DateTimeOffset? CompletedAt { get; private set; }
    public string? Error { get; private set; }

    public void Complete()
    {
        Status = JobStatus.Completed;
        CompletedAt = DateTimeOffset.UtcNow;
    }

    public void Fail(string error)
    {
        Status = JobStatus.Failed;
        Error = error;
        CompletedAt = DateTimeOffset.UtcNow;
    }
}

public enum JobStatus { Pending, Running, Completed, Failed }
```

**Что улучшили.** Создать *ScheduledJob* без обязательных данных теперь невозможно — конструктор этого не позволит. *Status* инициализируется в известное начальное состояние, а не остаётся `null` до первого присваивания. Сеттеры закрыты — состояние меняется только через явные методы с понятными именами (*Complete*, *Fail*), и поменять статус в обход этих методов нельзя. Заодно `string Status` заменён на `enum JobStatus` — об этом подробнее в правиле 3, но в одном классе нарушены сразу два правила, и решение тоже одно.

---

### Пример 2.2. *Corporation* в домене импорта — protected-конструктор без аргументов

В *Loyalty.Domain.Import.Corporation* — *protected*-конструктор без параметров, оставленный «для EF». Это распространённый паттерн, но он приводит к тому, что объект создаётся ORM в пустом состоянии, а поля инициализируются через рефлексию. Публичного смыслового конструктора нет вовсе:

```csharp
// Было: нет публичного конструктора с параметрами — непонятно как создать Corporation правильно
public class Corporation
{
    protected Corporation() { }   // для EF

    public long Id { get; private set; }
    public bool IsActive { get; private set; }

    public void SetActive(bool isActive)
    {
        IsActive = isActive;
    }
}
```

Разработчик, которому нужно создать новую корпорацию, не видит никакого публичного конструктора с параметрами. Либо использует рефлексию, либо добавляет публичный конструктор по умолчанию — и объект снова можно создать без *Id*.

Дополнительный звоночек — метод *SetActive(bool)*. Принимает примитив. Вызывающий код может написать `corporation.SetActive(false)` или `corporation.SetActive(true)` — с одинаковой синтаксической лёгкостью, но с противоположным смыслом. Это та самая «одержимость примитивами» из правила 3, привезённая в API простой setter-функции.

Стоит отметить ещё одну пограничную находку из той же кодовой базы: в *Loyalty.Domain.Corporations.Corporation* (другом, не путать с тем что выше) лежит **полностью пустой** класс — без полей, без методов, даже без конструктора. Это уже не «нет смыслового конструктора», а «нет вообще ничего». Такой класс лучше либо удалить, либо наполнить — оставлять как есть нельзя: читатель тратит время на «может, я что-то пропустил».

```csharp
// Стало: явный публичный конструктор + методы вместо примитивного setter-а
public class Corporation
{
    protected Corporation() { }   // для EF — пусть будет, но скрыт

    public Corporation(long id, bool isActive = true)
    {
        if (id <= 0)
            throw new ArgumentException("Id корпорации должен быть положительным", nameof(id));
        Id = id;
        IsActive = isActive;
    }

    public long Id { get; private set; }
    public bool IsActive { get; private set; }

    // Вместо SetActive(bool) — два явных метода с однозначной семантикой
    public void Activate()   => IsActive = true;
    public void Deactivate() => IsActive = false;
}
```

**Что улучшили.** Появился явный публичный конструктор — разработчик видит, что для создания корпорации нужен *Id*. EF-конструктор остался *protected* — случайно его не вызвать. *SetActive(bool)* заменён на два метода с понятными именами — нельзя передать неправильное значение, потому что значение вообще не передаётся. Создать корпорацию с `Id = 0` или отрицательным невозможно — конструктор это запрещает.

---

## Правило 3. Избегать примитивных типов, строить прикладную систему типов

Суть правила: строка или число не несут никакой информации о том, что в них можно или нельзя положить. Прикладной тип — это контракт: он описывает допустимые значения и делает неправильное использование заметным на этапе компиляции.

### Пример 3.1. *CalculationStatus* — строка вместо перечисления в ответах API

В нескольких DTO — *RefundCalculationDto*, *CloseCalculationDto* — статус расчёта возвращается как *string*:

```csharp
// Было: статус — строка, которую можно заполнить чем угодно
public class RefundCalculationDto
{
    public Guid Id { get; set; }
    public int Version { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }
    public string Status { get; set; }  // "active"? "Active"? "ACTIVE"? любое значение
    public List<RefundDish> RefundedDishes { get; set; }
    public List<RefundReceipt> RefundedReceipts { get; set; }

    public static RefundCalculationDto MapFrom(Calculation calculation)
    {
        return new RefundCalculationDto
        {
            Status = calculation.Status.ToString(),   // конвертация в строку — точка потери типа
            // ...
        };
    }
}
```

*Status* как *string* в DTO означает, что клиент получает строку и сам разбирается, что с ней делать. На стороне клиента — снова *string*, снова ручной парсинг, снова опечатки. Добавить новый статус — не забыть поменять все места с ручным сравнением строк. И главное: *CalculationStatus* как *enum* в домене **уже существует**. То есть мы намеренно теряем тип на границе DTO и заставляем потребителя выводить его обратно.

```csharp
// Стало: статус как enum в DTO тоже — клиент получает типизированное значение
public class RefundCalculationDto
{
    public Guid Id { get; set; }
    public int Version { get; set; }
    public DateTimeOffset UpdatedAt { get; set; }
    public CalculationStatus Status { get; set; }  // компилятор знает допустимые значения
    public List<RefundDish> RefundedDishes { get; set; }
    public List<RefundReceipt> RefundedReceipts { get; set; }

    public static RefundCalculationDto MapFrom(Calculation calculation)
    {
        return new RefundCalculationDto
        {
            Status = calculation.Status,   // прямое присваивание, без ToString() и без парсинга
            // ...
        };
    }
}
```

**Что улучшили.** *CalculationStatus* уже существует как *enum* в домене — нет смысла конвертировать его в строку только для того, чтобы потом снова парсить. Клиент получает типизированное значение, IDE подскажет допустимые варианты при `switch`-е, компилятор выдаст предупреждение, если добавится новый статус и его забудут обработать. Сериализация в JSON через `JsonStringEnumConverter` сохраняет читаемый текстовый вид на проводе — потери для внешнего клиента нет, а внутри C#-кода тип не теряется.

---

### Пример 3.2. *MenuProduct* — все числовые поля как *string*

В *MenuProduct* из интеграции с White Server — поля *Price*, *Protein*, *Fat*, *Carbohydrates*, *Kcal* объявлены как *string*:

```csharp
// Было: числовые характеристики блюда — строки
public class MenuProduct
{
    public string Id { get; set; }
    public string CategoryId { get; set; }
    public string Name { get; set; }
    public string Price { get; set; }       // "150.00"? "150,00"? ""? null?
    public string Protein { get; set; }     // "12.5"? null?
    public string Fat { get; set; }
    public string Carbohydrates { get; set; }
    public string Kcal { get; set; }
    // ...
}
```

Это не случайность — внешнее API действительно возвращает числа как строки (что само по себе плохой дизайн, но он чужой). Проблема в том, что этот же тип используется внутри системы. И здесь интересный нюанс: рядом, в *Loyalty.Domain.Menus.Product*, **уже существует** правильный доменный класс с типами `decimal Price`, `float Protein`, `float Fat`, `float Carbohydrates`, `float Kcal`. Архитектура понимает, что между ними должна быть граница — но эта граница плохо проведена.

В кодовой базе уже есть расширение *ConvertOrDefault\<T\>* для работы со строковыми числами — значит, проблема известна и обходится вручную в каждом месте использования. Это типичный «костыль вместо границы»: вместо того чтобы один раз превратить *MenuProduct* в *Product* на входе, мы носим строки по всей системе и парсим их там, где удобно.

```csharp
// Стало: парсинг происходит один раз — на границе с внешним API.
// Внутри системы используется уже существующий доменный Product — никаких строковых чисел.
public class MenuProduct
{
    // Этот класс остаётся «зеркалом» внешнего API — у него своя задача:
    // принять данные ровно в том виде, в каком их прислали.
    public string Id { get; set; }
    public string CategoryId { get; set; }
    public string Name { get; set; }
    public string Price { get; set; }
    public string Protein { get; set; }
    public string Fat { get; set; }
    public string Carbohydrates { get; set; }
    public string Kcal { get; set; }
    // ... + методы конвертации (или явный маппер) ниже
}

// Маппер, который превращает «зеркало» в доменный объект — единственная точка парсинга.
public static class MenuProductMapper
{
    public static Product ToDomain(MenuProduct dto, long corporationId, long restaurantId)
    {
        return new Product(
            corporationId,
            restaurantId,
            id:    long.Parse(dto.Id),
            name:  dto.Name,
            price: ParseDecimalOrZero(dto.Price),
            // ... protein, fat и т. д. — все парсятся здесь, и больше нигде
            protein:       ParseFloatOrZero(dto.Protein),
            fat:           ParseFloatOrZero(dto.Fat),
            carbohydrates: ParseFloatOrZero(dto.Carbohydrates),
            kcal:          ParseFloatOrZero(dto.Kcal),
            // ...
        );
    }

    private static decimal ParseDecimalOrZero(string s) =>
        decimal.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var v) ? v : 0m;

    private static float ParseFloatOrZero(string s) =>
        float.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var v) ? v : 0f;
}

// Внутри системы — никакого ConvertOrDefault больше нет.
// Везде, где раньше работали с MenuProduct, теперь работают с Product:
if (product.Price > 0) { /* нормальное сравнение decimal, не строк */ }
```

**Что улучшили.** Парсинг строк из внешнего API происходит один раз — на границе системы, в маппере. Внутри — только типизированные значения через уже существующий доменный *Product*. Нельзя случайно передать *"abc"* как цену и получить исключение в глубине бизнес-логики. Проверка на нулевую цену читается как нормальное сравнение чисел, а не как `price != null && price != "" && price != "0"`. Граница между «как нам прислали» и «как у нас принято» теперь проведена явно.

---

## Правило 1. Дополнительный пример — *GuestSegments.IsMatch* без предварительной инициализации

*GuestSegments* — условие, которое проверяет, входит ли гость в один из сегментов акции. Чтобы оно работало, его нужно инициализировать в **два шага**: сначала *InitBenefit* (передать *corporationId*, *benefitId*, *benefitType*), потом *LoadBenefitData* (загрузить *SegmentIds* из БД). Если пропустить любой из шагов и вызвать *IsMatch* — условие либо вернёт *false* (пустой список), либо упадёт внутри с *NullReferenceException*.

Это классическая ловушка двухшаговой инициализации. Разработчик создаёт условие, не знает, что нужно ещё два шага, вызывает *IsMatch* — и получает тихий *false* вместо явной ошибки. И это та самая «фигура зла», описанная в материале про шахматы: API позволяет вызвать методы в любом порядке, а защиты от неправильного порядка нет.

```csharp
// Было: три публичных метода, которые нужно вызвать в правильном порядке
public class GuestSegments : Condition, IBenefitAwareWithLoad
{
    public List<Guid> SegmentIds { get; set; }  // null пока не вызван LoadBenefitData

    // Шаг 1 — обязателен
    public void InitBenefit(long corporationId, Guid benefitId, BenefitType benefitType)
    {
        CorporationId = corporationId;
        BenefitId = benefitId;
        BenefitType = benefitType;
    }

    // Шаг 2 — обязателен, зависит от шага 1
    public async Task LoadBenefitData(long corporationId, Guid benefitId, BenefitType benefitType)
    {
        var segmentRepository = ConditionContext.GetRepository<ISegmentRepository>();
        SegmentIds = await segmentRepository
            .GetSegmentIdsByBenefitId(corporationId, benefitId, benefitType);
    }

    public override bool IsMatch(Calculation calculation)
    {
        // Если LoadBenefitData не вызвали — SegmentIds == null, тихий false
        if (SegmentIds == null || !SegmentIds.Any())
            return false;
        // ...
    }
}

// Вызывающий код может легко пропустить шаги:
var condition = new GuestSegments();
condition.InitDb(context);
// Забыли InitBenefit и LoadBenefitData — IsMatch тихо вернёт false
var result = condition.IsMatch(calculation);
```

```csharp
// Стало: нельзя получить GuestSegments без данных сегментов.
// Фабричный метод заменяет двухшаговую инициализацию.
public class GuestSegments : Condition
{
    private readonly IReadOnlyList<Guid> _segmentIds;

    // Конструктор приватный — внешний код не может создать недоинициализированный объект
    private GuestSegments(IReadOnlyList<Guid> segmentIds)
    {
        _segmentIds = segmentIds.Count > 0
            ? segmentIds
            : throw new ArgumentException("Список сегментов не может быть пустым", nameof(segmentIds));
    }

    // Единственный способ создать условие — через фабричный метод,
    // который сам выполняет все шаги инициализации
    public static async Task<GuestSegments> CreateAsync(
        long corporationId, Guid benefitId, BenefitType benefitType,
        ISegmentRepository repository)
    {
        var segmentIds = await repository.GetSegmentIdsByBenefitId(corporationId, benefitId, benefitType);
        return new GuestSegments(segmentIds);
    }

    public override bool IsMatch(Calculation calculation)
    {
        // _segmentIds гарантированно непустой — конструктор это обеспечил
        // ...
    }
}

// Вызывающий код:
// Забыть инициализацию невозможно — CreateAsync делает всё сам
var condition = await GuestSegments.CreateAsync(corporationId, benefitId, benefitType, repository);
var result = condition.IsMatch(calculation);
```

**Что улучшили.** Двухшаговая инициализация заменена единственным фабричным методом. Нельзя создать *GuestSegments* с пустым списком сегментов — конструктор это запрещает. Нельзя забыть загрузить данные — *CreateAsync* делает это как часть создания объекта. Тихий *false* из-за пропущенного шага больше невозможен. Это ровно тот переход, который описан в материале: «избавиться от явных шагов, инкапсулировать состояние внутрь».

---

## Правило 2. Дополнительный пример — *Discount* в домене: internal-конструктор и публичные сеттеры коллекций

*Loyalty.Domain.Discounts.Discount* — доменная сущность скидки. Открываешь и видишь *protected Discount() { }* — пустой конструктор для EF. Дальше есть *internal* конструктор с параметрами — но он **internal**, то есть доступен только сборке *Loyalty.Domain*. С точки зрения внешнего слоя (инфраструктуры, сервисов в других сборках) полноценного публичного API для создания скидки нет — а сеттеры на коллекциях открыты:

```csharp
// Было: конструктор с параметрами есть, но он internal —
// внешним сборкам недоступен. И при этом сеттеры на Conditions/Reward — public.
public class Discount
{
    protected Discount() { }  // для EF

    internal Discount(long corporationId, string name, string description, bool isActive,
        List<Condition> conditions, Reward reward)
    {
        // ... корректная инициализация
    }

    public long CorporationId { get; private set; }
    public Guid Id { get; private set; }
    public bool IsCombined { get; } = true;
    public string Name { get; private set; }
    public string Description { get; private set; }
    public bool IsActive { get; private set; }

    public List<Condition> Conditions { get; set; } = new();   // ← публичный set
    public Reward Reward { get; set; }                          // ← публичный set

    public bool? IsDeleted { get; private set; }
    public DateTimeOffset CreateDate { get; private set; }
    public DateTimeOffset? UpdateDate { get; private set; }
}

// Что позволяет делать такой дизайн:
// 1) Из любой сборки, где есть InternalsVisibleTo — создать через internal-конструктор.
// 2) Из любой сборки — создать через рефлексию protected-конструктор и заполнить руками:
var discount = (Discount)Activator.CreateInstance(typeof(Discount), nonPublic: true);
discount.Conditions = new List<Condition>();   // без Reward
discount.Reward = null;                         // скидка без награды — бессмысленно, но компилируется
// 3) Даже после правильного конструктора — кто угодно может перезаписать Reward на null.
existingDiscount.Reward = null;
```

```csharp
// Стало: явный публичный конструктор + сеттеры закрыты.
// Изменение коллекций условий — только через явные методы.
public class Discount
{
    protected Discount() { }  // для EF — оставляем, но он скрыт

    public Discount(
        long corporationId,
        string name,
        string description,
        bool isActive,
        bool isCombined,
        List<Condition> conditions,
        Reward reward)
    {
        if (string.IsNullOrWhiteSpace(name))
            throw new ArgumentException("Название скидки обязательно", nameof(name));
        if (reward == null)
            throw new ArgumentNullException(nameof(reward), "Скидка должна иметь награду");

        Id = Guid.NewGuid();
        CorporationId = corporationId;
        Name = name;
        Description = description;
        IsActive = isActive;
        IsCombined = isCombined;
        Conditions = conditions ?? new List<Condition>();
        Reward = reward;
        CreateDate = DateTimeOffset.UtcNow;
    }

    public long CorporationId { get; private set; }
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public string Description { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsCombined { get; private set; }
    public List<Condition> Conditions { get; private set; }   // set закрыт
    public Reward Reward { get; private set; }                // set закрыт

    // Изменения — через явные методы с проверками
    public void Replace(string name, string description, bool isActive, bool isCombined,
                        List<Condition> conditions, Reward reward)
    {
        if (reward == null)
            throw new ArgumentNullException(nameof(reward));
        Name = name; Description = description; IsActive = isActive; IsCombined = isCombined;
        Conditions = conditions ?? new List<Condition>();
        Reward = reward;
    }
}
```

**Что улучшили.** Скидку без названия или без награды создать невозможно — конструктор не пропустит. *Conditions* и *Reward* больше не имеют публичных сеттеров — изменить их можно только через явный *Replace* с такими же проверками. Существующая скидка не может вдруг оказаться без награды — присвоение `discount.Reward = null` перестало компилироваться. EF-конструктор остался *protected* — случайно не вызвать.

---

## Правило 3. Дополнительный пример — *BenefitType* как строка в SQL-запросах

*BenefitType* — перечисление с тремя значениями: *Promotion*, *Promocode*, *Discount*. В домене оно правильно объявлено как *enum*. Но в инфраструктурном слое при работе с БД это значение конвертируется в строку через *ToString()* прямо в параметрах SQL-запросов:

```csharp
// Было: BenefitType преобразуется в строку прямо в месте использования
await connection.ExecuteAsync(deleteSql, new
{
    benefitId,
    corporationId,
    benefitType = benefitType.ToString()  // "Promotion", "Promocode" или "Discount"
});

// И снова в другом запросе:
var records = segmentCondition.SegmentIds.Select(segmentId => new
{
    corporationId,
    benefitId,
    benefitType = benefitType.ToString(),  // дублирование конвертации
    segmentId
});
```

Проблема не в самом *ToString()* — это мелочь. Проблема в том, что строковое представление в БД и значение *enum* в коде теперь живут независимо. Если переименовать `BenefitType.Promocode` в `BenefitType.PromoCode` (с большой буквой) — рефакторинг в IDE переименует enum, но не тронет данные в БД. Запросы перестанут матчиться, и ошибка проявится только в рантайме на конкретных данных.

```csharp
// Стало: конвертация один раз в одном месте — через TypeHandler у Dapper.

public class BenefitTypeHandler : SqlMapper.TypeHandler<BenefitType>
{
    // Контракт «как BenefitType хранится в БД» зафиксирован здесь
    public override void SetValue(IDbDataParameter parameter, BenefitType value)
        => parameter.Value = value switch
        {
            BenefitType.Promotion => "Promotion",
            BenefitType.Promocode => "Promocode",
            BenefitType.Discount  => "Discount",
            _ => throw new ArgumentOutOfRangeException(nameof(value), value, "Неизвестный тип")
        };

    public override BenefitType Parse(object value) => (string)value switch
    {
        "Promotion" => BenefitType.Promotion,
        "Promocode" => BenefitType.Promocode,
        "Discount"  => BenefitType.Discount,
        _ => throw new ArgumentException($"Неизвестное значение BenefitType в БД: {value}")
    };
}

// Регистрация однократно при старте приложения:
SqlMapper.AddTypeHandler(new BenefitTypeHandler());

// После регистрации TypeHandler конвертация полностью исчезает из кода:
await connection.ExecuteAsync(deleteSql, new
{
    benefitId,
    corporationId,
    benefitType   // Dapper сам сериализует BenefitType в нужную строку
});
```

**Что улучшили.** Контракт «как *BenefitType* хранится в БД» зафиксирован в одном месте, а не размазан по SQL-запросам через *ToString()*. Переименование значения *enum* не сломает данные в БД — изменить нужно только *BenefitTypeHandler*. Попытка передать несуществующее значение — исключение сразу, не тихая потеря данных при выборке. Места использования становятся проще — параметр передаётся как обычный enum, без ручных конверсий.

---

## Общая рефлексия

Три правила выглядят как технические советы, но за каждым стоит одна и та же идея: ошибка разработчика — это почти всегда ошибка проектировщика. Если джуниор вызвал *Close* у расчёта без гостя — не потому что невнимательный, а потому что интерфейс позволил. Если кто-то создал *ScheduledJob* без *Id* — не потому что не знал, а потому что конструктор не потребовал.

Самый интересный момент из работы над примерами: в каждом случае «защита» уже существовала — но в виде проверки в сервисе, комментария в коде или документации. Правило говорит сделать шаг дальше: перенести эту защиту на уровень интерфейса класса так, чтобы нарушение стало физически невозможным. Разница между «здесь написано, что нельзя» и «компилятор не даст» — огромная.

Заметна закономерность: правила пересекаются. *ScheduledJob* нарушает сразу два — нет конструктора (правило 2) и `Status` как строка (правило 3). *Discount* нарушает второе и формально первое — публичные сеттеры коллекций позволяют привести объект в недопустимое состояние. *MenuProduct* — третье, но через костыль *ConvertOrDefault* фактически становится и первым (типы парсятся в случайных местах, и любая неудача парсинга — бомба замедленного действия). Это не случайность: все три правила про одно и то же — про то, чтобы интерфейс класса был честным относительно того, какие состояния и операции в нём допустимы.

Первое правило оказалось самым ёмким, потому что охватывает не только порядок вызовов, но и состояния объекта. Хорошо спроектированный класс не может оказаться в недопустимом состоянии — не потому что разработчик был осторожен, а потому что такое состояние просто не существует в системе типов.