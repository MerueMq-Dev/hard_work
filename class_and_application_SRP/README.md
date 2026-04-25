# Отчёт: Плохой стиль на уровне классов и приложения

## Введение

Задание делит проблемы на два уровня — классов и приложений. Граница между ними важная и не косметическая: она про то, где проблема видна.

На уровне классов проблема локализована в одном файле. Открыл *UpdateMenuCorporationSaga*, увидел сорок полей — всё, симптом перед глазами. На уровне приложения проблема рассыпана по нескольким файлам, и в каждом из них по отдельности всё «нормально». Чтобы её заметить, нужно держать в голове сразу три-четыре места, которые при чтении кода никто рядом не открывает. Поэтому проблемы уровня приложения дороже: они дольше живут незамеченными и накапливают цену каждый раз, когда новый разработчик "логично" добавляет ещё одну копию.

При этом две категории не изолированы. Дублирование полей в *CalcDishDto*/*CalcIngredientDto* (2.7) и дублирование полей в *Create/EditPromotionRequest* (3.1) — это одна и та же проблема, увиденная в разных масштабах: в первом случае — внутри одной иерархии типов, во втором — между параллельными ветками приложения. Поэтому в отчёте видно, как одни и те же приёмы (выделение общей концепции, отказ от копирования) работают на обоих уровнях.

Первая реакция на половину примеров — «ну и что, работает же». Класс большой — зато всё в одном месте. Класс пустой — ну, добавят потом. Метод не там — подумаешь, мелочь. Но именно здесь прячутся проблемы, которые потом не гасятся за вечер. Строку правишь за минуту. Класс, который за два года оброс сорока полями и десятком методов из разных слоёв — это уже разговор на несколько спринтов.

---

## 2.1. Класс слишком большой или в программе создаётся слишком много его инстансов

В формулировке задания два случая стоят рядом не случайно: оба — про то, что у класса нечётко определена роль. В первом случае класс пытается быть всем сразу, во втором — наоборот, описывает что-то слишком мелкое, и поэтому размножается. Симптомы выглядят противоположно, а проблема одна: концепция выбрана неверно.

### Случай А — большой класс, нарушение SRP

Открываешь *UpdateMenuCorporationSaga* и сначала кажется, что всё нормально — ну сага, ну много полей. Потом начинаешь считать: идентификация, время создания, текущее состояние, описание состояния, четыре счётчика прогресса, список ресторанов, текущая команда для отправки, токен конкурентности. И ещё методы — сага сама решает, какой ресторан обновлять следующим, сама считает проценты, сама формирует команды.

Это не сага. Это три разных объекта, случайно приклеенных друг к другу: модель состояния, счётчик прогресса и планировщик очереди. Когда надо поменять формулу расчёта процентов — лезешь в файл с логикой обхода ресторанов. Когда надо протестировать выбор следующего ресторана — поднимаешь весь граф саги.

### Было

```csharp
public class UpdateMenuCorporationSaga : SagaStateMachineInstance
{
    // Идентификация и время
    public Guid CorrelationId { get; set; }
    public DateTime Created { get; private set; }
    public DateTime Updated { get; private set; }
    public long CorporationId { get; private set; }

    // Состояние саги
    public string State { get; set; }
    public string StateDescription { get; private set; }

    // Прогресс — четыре разных счётчика
    public int TotalSteps { get; private set; }
    public int CompleteSteps { get; private set; }
    public int PercentOfComplete { get; private set; }
    public int TotalRestaurantCount { get; private set; }
    public int CompleteRestaurantCount { get; private set; }

    // Список ресторанов и текущая команда
    public List<MenuRestaurantItem> MenuRestaurantStates { get; private set; }
    public ICommand Command { get; private set; }
    public Guid ConcurrencyToken { get; private set; }

    // Методы — сага сама выбирает следующий ресторан
    public void UpdateMenuForRestaurants() { ... }
    public void RestaurantUpdateSuccess(long id) { ... }
    public void RestaurantUpdateFailed(long id, ...) { ... }

    private bool FindRestaurantForUpdate() { ... }
    private void IncrementProgress() { ... }
    private void IncrementProgressForRestaurant() { ... }
    private void CheckCorporationUpdateLoaded() { ... }
}
```

### Стало

```csharp
// Прогресс — отдельная концепция
public class SagaProgress
{
    public int TotalSteps { get; private set; }
    public int CompleteSteps { get; private set; }
    public int PercentOfComplete { get; private set; }
    public int TotalRestaurantCount { get; private set; }
    public int CompleteRestaurantCount { get; private set; }

    public void IncrementRestaurant()
    {
        CompleteRestaurantCount++;
        CompleteSteps++;
        PercentOfComplete = TotalSteps > 0 ? (100 * CompleteSteps) / TotalSteps : 0;
    }
}

// Список ресторанов и логика выбора следующего — отдельная концепция
public class RestaurantUpdateQueue
{
    private readonly List<MenuRestaurantItem> _items;

    public (MenuRestaurantItem? next, ICommand? command) TakeNext(Guid correlationId, long corporationId) { ... }
    public void MarkSuccess(long restaurantId) { ... }
    public void MarkFailed(long restaurantId, string error) { ... }
}

// Сама сага — координирует, но не считает прогресс и не выбирает ресторан
public class UpdateMenuCorporationSaga : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string State { get; set; }
    public SagaProgress Progress { get; private set; }
    public RestaurantUpdateQueue Queue { get; private set; }
    public ICommand Command { get; private set; }
    public Guid ConcurrencyToken { get; private set; }
}
```

Теперь поменять формулу расчёта процентов — это *SagaProgress*, не трогая ничего рядом. Протестировать выбор следующего ресторана — это *RestaurantUpdateQueue* в изоляции. Каждая концепция знает своё дело и не лезет в чужое.

### Случай Б — слишком много инстансов одного класса

Обратная ситуация. Если в системе на один запрос создаётся очень много экземпляров одного и того же класса, и каждый отличается от соседнего только одним-двумя значениями, — это сигнал, что класс описывает не концепцию, а отдельное значение.

В нашей кодовой базе так ведёт себя *DiscountedDish* — отдельный объект на каждую позицию заказа в каждом расчёте. На один заказ из десяти блюд получается десять *DiscountedDish* верхнего уровня плюс по одному на каждый ингредиент и модификатор внутри — реально 30–50 экземпляров на расчёт. Поля у *DiscountedDish* почти один в один копируют поля *Dish* (*LineItemId*, *Id*, *Name*, *Quantity*, *Price*, *Amount*, *Restrictions*, вложенные *Dishes*) плюс две скидочные величины: *Discount* и словарь *Discounts*. По смыслу это пара (исходное блюдо, начисленная на него скидка) — то есть строка таблицы скидок, привязанная к позиции. Но раз это класс с поведением, на него начинают вешать методы: *MergeDiscount*, *ClearDiscounts*, фабричный *CreateWithDiscount*. И каждый такой метод нужно тестировать на 30 разных инстансах в каждом сценарии. Вместо «у блюда X начислена скидка Y» появляется граф маленьких объектов, между которыми разбегается логика.

### Было

```csharp
// Класс существует ради одного дополнительного поля Discount поверх Dish
public class DiscountedDish
{
    // Эти 8 полей — копия из Dish
    public Guid LineItemId { get; set; }
    public long Id { get; }
    public string Name { get; }
    public decimal Quantity { get; }
    public int Price { get; }
    public long Amount { get; }
    public Restrictions Restrictions { get; }
    public bool IsDiscountExcluded { get; }
    public List<DiscountedDish> Dishes { get; }

    // А вот это — то, ради чего класс действительно нужен:
    public long Discount { get; set; }
    public Dictionary<string, long> Discounts { get; set; } = new();

    public void MergeDiscount(DiscountedDish other) { ... }
    internal void ClearDiscounts() { ... }
    private void WithDiscount(DiscountTypeEnum type, long discount) { ... }
}

// Расчёт держит параллельную коллекцию: на каждый Dish создаётся свой DiscountedDish
internal void InitDiscountedDishes()
{
    DiscountedDishes = OriginalDishes.Select(x =>
        new DiscountedDish(x.LineItemId!.Value, x.Id, x.Name, x.Quantity, x.Price, x.Amount, x.Restrictions,
            (x.Dishes != null) ? x.Dishes.Select(i => new DiscountedDish(...)).ToList() : null,
            x.IsDiscountExcluded)).ToList();
    // 30+ экземпляров на расчёт
}
```

### Стало

```csharp
// Скидка отделяется от блюда: блюдо остаётся блюдом, скидка — это маленькое значение,
// привязанное к LineItemId. Объектов с поведением больше не плодим.
public readonly record struct DiscountAllocation(
    Guid LineItemId,
    long Amount,
    IReadOnlyDictionary<string, long> ByType);

// Расчёт держит словарь скидок по позициям — одна структура вместо параллельной иерархии.
// Поведение (слияние, очистка) — методы расчёта, а не каждой позиции.
public class Calculation
{
    private readonly Dictionary<Guid, DiscountAllocation> _discounts = new();

    public IReadOnlyDictionary<Guid, DiscountAllocation> Discounts => _discounts;

    internal void MergeDiscount(Guid lineItemId, DiscountTypeEnum type, long amount) { ... }
    internal void ClearDiscounts() => _discounts.Clear();
}
```

Теперь *Dish* — это блюдо, а скидка — это аллокация на позицию. Тестировать поведение можно в одном месте, на расчёте, а не на каждом из тридцати *DiscountedDish*. И когда видишь, что на запрос создаются десятки инстансов одного класса с почти одинаковыми полями — стоит спросить: это объект со своим поведением, или это значение, которое мы зачем-то нарядили в класс?

---

## 2.2. Класс слишком маленький или делает слишком мало

Два случая рядом, и оба неприятные по-своему.

Первый — *Corporation* в *Loyalty.Domain.WebMan*. Открываешь файл и видишь класс из двух геттеров (*Id*, *IsActive*) — ни конструктора, ни одного метода, ни комментария зачем он вообще существует. По сути — анемичный DTO, замаскированный под доменный класс. Если завтра понадобится логика «активная корпорация может создавать заказы» — её некуда положить, и она расползётся по сервисам и хелперам. А пока класс просто занимает место и заставляет читателя гадать, доменная это сущность или transport-объект.

Второй — *AppliedPromocode*. Уже лучше: есть два поля, есть конструктор. Но конструктор принимает *Promocode* и... ничего с ним не делает. Поля остаются незаполненными. Это уже не просто пустота — это ловушка. Код компилируется, объект создаётся, но *Name* и *Amount* будут *null* или *0* в рантайме.

### Было

```csharp
// Loyalty.Domain.WebMan.Corporation.cs — два поля и больше ничего:
// нет конструктора, нет методов, нет инвариантов
public class Corporation
{
    protected Corporation() { }

    public long Id { get; private set; }

    public bool IsActive { get; private set; }
}
```

```csharp
public class AppliedPromocode
{
    public string Name { get; private set; }
    public decimal Amount { get; private set; }

    // Конструктор принимает Promocode, но не использует его —
    // поля Name и Amount так и не инициализируются
    internal AppliedPromocode(Promocode promocode)
    {
        // тело пустое
    }
}
```

### Стало

```csharp
// Вариант 1: если Corporation действительно нужна только как «id + флаг активности»,
// которые читаются из БД — это не доменная сущность, а DTO. Тогда честно перенести её
// в слой инфраструктуры под именем CorporationDto и не путать читателя.

// Вариант 2: если это всё-таки доменная сущность — дать ей минимальный набор
// инвариантов и поведения, ради которых класс вообще существует:
public class Corporation
{
    protected Corporation() { } // для ORM

    public long Id { get; private set; }
    public bool IsActive { get; private set; }

    public Corporation(long id)
    {
        Id = id;
        IsActive = true;
    }

    public void Activate()   => IsActive = true;
    public void Deactivate() => IsActive = false;
}

// AppliedPromocode — доделать конструктор:
internal AppliedPromocode(Promocode promocode)
{
    Name = promocode.Name;
    Amount = promocode.Amount;
}
```

Класс без поведения и без инвариантов — это либо DTO, который не на своём месте, либо незаконченная сущность. В обоих случаях нужно что-то сделать, а не оставлять как есть в надежде что «потом разберёмся».

---

## 2.3. Метод, который выглядит подходящим для другого класса

В *CalculationBotExtensions* живёт метод *GetBotRecipients*. Он берёт *Calculation* и перебирает *TelegramUserId*, *MaxUserId*, *VkUserId* — чтобы понять, каким ботам слать уведомление. Выглядит невинно, но если задуматься: при чём тут расчёт? Расчёт — это про скидки, блюда и бонусы. То, на каких платформах зарегистрирован гость — это знание о госте, не о расчёте. Расчёт здесь просто случайный носитель чужих данных.

Это ещё и практическая проблема: если добавить новую платформу, например WhatsApp — нужно лезть в код расчёта. Что само по себе странно.

### Было

```csharp
// CalculationBotExtensions.cs — метод живёт у расчёта
public static class CalculationBotExtensions
{
    public static IEnumerable<(BotPlatform Platform, long ChatId)> GetBotRecipients(
        this Calculation calculation)
    {
        // Расчёт перебирает свои поля и знает о платформах ботов.
        // Но откуда расчёт знает про Max, Telegram, Vk?
        if (calculation.MaxUserId != null)
            yield return (BotPlatform.Max, calculation.MaxUserId.Value);

        if (calculation.TelegramUserId != null)
            yield return (BotPlatform.Telegram, calculation.TelegramUserId.Value);

        if (calculation.VkUserId != null)
            yield return (BotPlatform.Vk, calculation.VkUserId.Value);
    }
}
```

### Стало

```csharp
// Метод переезжает в Guest — именно гость знает, на каких платформах он зарегистрирован
public class Guest
{
    public long? TelegramUserId { get; private set; }
    public long? MaxUserId { get; private set; }
    public long? VkUserId { get; private set; }

    // Теперь расчёт просто делегирует гостю:
    // calculation.Guest.GetBotRecipients()
    public IEnumerable<(BotPlatform Platform, long ChatId)> GetBotRecipients()
    {
        if (MaxUserId != null)
            yield return (BotPlatform.Max, MaxUserId.Value);

        if (TelegramUserId != null)
            yield return (BotPlatform.Telegram, TelegramUserId.Value);

        if (VkUserId != null)
            yield return (BotPlatform.Vk, VkUserId.Value);
    }
}
```

Теперь добавление WhatsApp — это изменение в *Guest*. Расчёт об этом вообще не знает.

---

## 2.4. Класс хранит данные, которые загоняются в него в множестве разных мест

*Calculation.Guest* — публичный сеттер. Казалось бы, удобно. Но это значит, что кто угодно может установить гостя когда угодно. И действительно устанавливает: в *CalculationRepository.Get* — `calculation.Guest = guest`, в обработчике запроса при загрузке расчёта — снова присваивание через сеттер, в тестовом маппере — через *AutoMapper*. Три места, одно поле.

Хуже всего, что в том же *Calculation* уже **существует** метод *SetGuest(guest, bonusesAccrualDisabled, antifraudRules)* — он привязывает гостя и заодно подтягивает связанные данные (телефон, имя, идентификаторы ботов). И в каких-то местах кода используется он, а в каких-то — прямой сеттер. То есть в коде сосуществуют два способа сделать одно и то же. Это уже не «не закрыли инкапсуляцию» — это «инкапсуляцию начали закрывать, но бросили на полпути». Если где-то установили не то или не вовремя — ошибку искать неприятно вдвойне: непонятно, кто из источников виноват, и непонятно, должно ли вообще здесь быть присваивание или вызов метода.

### Было

```csharp
// CalculationRepository.cs — присваивание через сеттер
calculation.Guest = guest;
calculation.Receipts.AddRange(receipts);

// В обработчике запроса при загрузке расчёта — то же присваивание
calculation.Guest = guest;

// AutoMapper в тестах
cfg.CreateMap<CalculationDto, Calculation>()
   .ForMember(x => x.Guest, ...);

// А в Calculation одновременно есть «правильный» метод, который тоже используется в части кода:
internal void SetGuest(Guest guest, bool bonusesAccrualDisabled, List<TriggeredRule> antifraudRules)
{
    GuestPhone = guest?.Phone;
    GuestId = guest?.Id;
    TelegramUserId = guest?.TelegramUserId;
    MaxUserId = guest?.MaxUserId;
    VkUserId = guest?.VkUserId;
    GuestName = guest?.TelegramUser?.FirstName ?? ...;
    // ... плюс антифрод, бонусы, сообщения
}
// → два пути привязки гостя сосуществуют, и часть данных (телефон, имя, идентификаторы ботов)
//   подтянется только если использовать метод
```

### Стало

```csharp
public class Calculation
{
    // Сеттер закрыт — единственный путь через AttachGuest
    public Guest Guest { get; private set; }

    // Единственная точка входа для привязки гостя.
    // Берёт на себя всё, что раньше делал SetGuest, плюс инвариант «один раз»
    internal void AttachGuest(Guest guest, bool bonusesAccrualDisabled, List<TriggeredRule> antifraudRules)
    {
        if (Guest != null)
            throw new InvalidOperationException("Гость уже привязан к расчёту");

        Guest = guest ?? throw new ArgumentNullException(nameof(guest));
        GuestPhone = guest.Phone;
        GuestId = guest.Id;
        TelegramUserId = guest.TelegramUserId;
        MaxUserId = guest.MaxUserId;
        VkUserId = guest.VkUserId;
        GuestName = guest.TelegramUser?.FirstName ?? ...;
        // ... антифрод, бонусы, сообщения — всё в одном месте
    }
}

// Все места привязки гостя — теперь одинаковые
calculation.AttachGuest(guest, bonusesAccrualDisabled, antifraudRules);
```

Теперь способ ровно один. Прямого сеттера нет — конкурирующего пути не появится. Попытка привязать второй раз — исключение, не молчаливая перезапись и не «забыли подтянуть телефон». Найти кто и когда привязал гостя — один поиск по *AttachGuest*.

---

## 2.5. Класс зависит от деталей реализации других классов

*ForFirstOrder* — условие «за первый заказ». Звучит просто. Но внутри *IsMatch* происходит вот что: условие лезет в *ConditionContext*, достаёт из него репозиторий через дженерик-метод *GetRepository\<T\>*, вызывает асинхронный *HasOrder* и блокирует поток через *GetAwaiter().GetResult()*. Условие знает не только что спросить, но и как именно устроен контекст изнутри, и что запрос асинхронный. Это уже не зависимость от контракта — это зависимость от реализации.

Тут две беды одновременно: семантическая (условие лезет в потроха контекста) и техническая (блокирующий вызов асинхронного кода — это путь к дедлокам и сожранным потокам). Просто заменить *GetRepository* на готовый метод контекста — половина решения; асинхронность всё равно остаётся, и кто-то снова напишет *GetAwaiter().GetResult()*. Лучше избавиться и от того, и от другого: данные, нужные условию, готовятся заранее, а *IsMatch* остаётся честно синхронным.

### Было

```csharp
public class ForFirstOrder : Condition
{
    public override bool IsMatch(Calculation calculation)
    {
        // Знает, как достать репозиторий из контекста (деталь реализации IConditionContext)
        var repository = ConditionContext.GetRepository<IConditionRepository>();

        // Знает, что метод асинхронный и блокирует поток вручную
        var hasOrder = repository.HasOrder(calculation).GetAwaiter().GetResult();

        return !hasOrder;
    }
}
```

### Стало

```csharp
// Контекст несёт уже подготовленные данные о госте — синхронный снимок,
// собранный один раз до начала проверки условий
public interface IConditionContext
{
    GuestSnapshot Guest { get; }
}

public sealed record GuestSnapshot(bool HasPreviousOrders, IReadOnlyCollection<long> SegmentIds);

public class ForFirstOrder : Condition
{
    // Условие задаёт бизнес-вопрос — «это первый заказ?» — и получает синхронный ответ.
    // Не знает ни про репозиторий, ни про async/await
    public override bool IsMatch(Calculation calculation)
    {
        return !ConditionContext.Guest.HasPreviousOrders;
    }
}

// Снимок собирается заранее, в одном асинхронном месте, до прогона условий
public async Task<GuestSnapshot> BuildSnapshotAsync(Calculation calculation)
{
    var hasOrders = await _repository.HasOrder(calculation);
    var segments = await _repository.GetSegments(calculation.Guest.Id);
    return new GuestSnapshot(hasOrders, segments);
}
```

Теперь *ForFirstOrder* действительно ничего не знает про устройство контекста — ни про репозитории, ни про асинхронность. И что важно — *GetAwaiter().GetResult()* исчезает физически, а не маскируется внутрь чужого метода. Все асинхронные обращения к БД делаются один раз, до начала проверки условий, а сама проверка остаётся быстрой и синхронной.

---

## 2.6. Приведение типов вниз по иерархии

*ConditionFactory.Create* возвращает *Condition*. Казалось бы, всё правильно — базовый тип, полиморфизм. Но потом в коде хэндлеров появляется `if (condition is IBenefitAwareWithLoad benefitAware)` — приведение к специальному интерфейсу-маркеру для тех условий, которым нужно дополнительно подгрузить данные из БД. И таких мест **три** — в хэндлерах *Promotions*, *Promocodes*, *Discounts* — буквально одинаковый код после получения условия из фабрики.

Формально это не приведение «к более узкому классу», а проверка на интерфейс. Но симптом тот же: чтобы корректно использовать объект, вызывающему коду нужно знать про его подтип. Полиморфизм недоделан — фабрика сделала вид, что отдаёт абстракцию, а на самом деле часть условий требует дополнительного шага, и про этот шаг знает каждый хэндлер. Если завтра появится третий уровень («условие, которое требует загрузки и дополнительной валидации»), все три хэндлера придётся править одинаково.

### Было

```csharp
// ConditionFactory возвращает базовый тип
public Condition Create(string name, dynamic data)
{
    string typeFullName = $"{_namespacestring}.{name}";
    Condition result = JsonSerializer.Deserialize(data, Type.GetType(typeFullName), ...);
    result.InitDb(_conditionContext);
    return result;
}

// Promotions/RequestHandler.cs — хэндлер вынужден знать про IBenefitAwareWithLoad
foreach (var c in conditionsDao)
{
    var condition = _conditionFactory.Create(c.name, c.data);
    if (condition is IBenefitAwareWithLoad benefitAware)
        await benefitAware.LoadBenefitData(corporationId, result.Id, BenefitType.Promotion);

    result.Conditions.Add(new ConditionDto { Name = c.name, Params = condition });
}

// Promocodes/RequestHandler.cs — тот же код, только BenefitType другой
if (condition is IBenefitAwareWithLoad benefitAware)
    await benefitAware.LoadBenefitData(corporationId, result.Id, BenefitType.Promocode);

// Discounts/RequestHandler.cs — снова то же самое
if (condition is IBenefitAwareWithLoad benefitAware)
    await benefitAware.LoadBenefitData(corporationId, result.Id, BenefitType.Discount);
```

### Стало

```csharp
// Базовый класс получает виртуальный метод инициализации.
// Условиям, которым нечего загружать — реализация по умолчанию пустая.
// Условиям с подгрузкой (как GuestSegments) — переопределяют и делают своё.
public abstract class Condition
{
    public virtual Task InitializeAsync(IConditionContext context,
                                        long corporationId, Guid benefitId, BenefitType benefitType)
        => Task.CompletedTask;

    public abstract bool IsMatch(Calculation calculation);
}

public class GuestSegments : Condition
{
    public List<Guid> SegmentIds { get; private set; }

    public override async Task InitializeAsync(IConditionContext context,
                                                long corporationId, Guid benefitId, BenefitType benefitType)
    {
        var repo = context.GetRepository<ISegmentRepository>();
        SegmentIds = await repo.GetSegmentIdsByBenefitId(corporationId, benefitId, benefitType);
    }
}

// Фабрика создаёт условие и сразу его инициализирует.
// Никакой is/as на стороне вызывающего кода больше не нужен.
public async Task<Condition> CreateAsync(string name, dynamic data,
                                          long corporationId, Guid benefitId, BenefitType benefitType)
{
    string typeFullName = $"{_namespacestring}.{name}";
    Condition result = JsonSerializer.Deserialize(data, Type.GetType(typeFullName), ...);
    await result.InitializeAsync(_conditionContext, corporationId, benefitId, benefitType);
    return result;
}

// Все три хэндлера — одинаковая короткая строка, без проверок типа
foreach (var c in conditionsDao)
{
    var condition = await _conditionFactory.CreateAsync(c.name, c.data,
                                                         corporationId, result.Id, BenefitType.Promotion);
    result.Conditions.Add(new ConditionDto { Name = c.name, Params = condition });
}
```

Правило простое: если после получения объекта из фабрики нужно его приводить к конкретному типу или интерфейсу-маркеру — значит, нужная логика должна жить в базовом контракте или внутри самой фабрики, не снаружи. Здесь оба требования закрыты сразу: контракт расширен виртуальным методом, а фабрика сама вызывает инициализацию.

Это решение делает фабрику асинхронной — это явный архитектурный trade-off. Цена: цепочка вызовов от хэндлера до фабрики становится async. Альтернатива — двухфазное создание (синхронный *Create*, отдельный синхронный *InitializeAll(conditions)*-метод на стороне сервиса) — выглядит проще, но возвращает прежнюю проблему: про вторую фазу можно забыть в новом месте использования. Async-фабрика хуже по эргономике, но безопаснее по семантике: невозможно получить из неё неинициализированный объект.

---

## 2.7. При создании наследника приходится создавать наследников и для других классов

*CalcDishDto*, *CalcIngredientDto*, *CalcModificatorDto* — три класса, описывающих позиции заказа на разных уровнях иерархии: блюдо, ингредиент, модификатор. Открываешь каждый и видишь одни и те же восемь полей: *LineItemId*, *Id*, *Name*, *Quantity*, *Price*, *Amount*, *MinPrice*, *MaxPrice*. Отличается только тип вложенной коллекции *Dishes*.

Хуже того: в проекте рядом параллельно живёт ещё одна такая же троица — *DishDto*, *IngredientDto*, *ModificatorDto* в *Loyalty.WebApi.Dto.Calculations*, для приёма данных из HTTP-запросов. То есть копий не три, а **шесть** в двух разных слоях. Симптом тот же — добавление поля порождает каскад правок — но масштаб больше.

Проблема проявляется при изменении. Добавить поле *VatRate* для налогов — три файла в инфраструктуре, и если симметрично нужно принять его с фронта — ещё три в WebApi. Добавить новый уровень иерархии, например *CalcComboDto* — снова восемь полей с нуля плюс новый файл, и сразу два таких файла, если общая идея распространяется на оба слоя. Каждое расширение стоит в шесть раз дороже, чем должно.

### Было

```csharp
// Три класса, три раза одинаковые поля
public class CalcDishDto
{
    public Guid? LineItemId { get; set; }
    public long Id { get; set; }
    public string Name { get; set; }
    public decimal Quantity { get; set; }
    public int Price { get; set; }
    public int Amount { get; set; }
    public int? MinPrice { get; set; }
    public int? MaxPrice { get; set; }
    public IEnumerable<CalcIngredientDto> Dishes { get; set; }  // ← отличие только здесь
    public bool? IsDiscountExcluded { get; set; }
}

public class CalcIngredientDto
{
    public Guid? LineItemId { get; set; }
    public long Id { get; set; }
    public string Name { get; set; }
    public decimal Quantity { get; set; }
    public int Price { get; set; }
    public int Amount { get; set; }
    public int? MinPrice { get; set; }
    public int? MaxPrice { get; set; }
    public IEnumerable<CalcModificatorDto> Dishes { get; set; }  // ← отличие только здесь
}

public class CalcModificatorDto
{
    public Guid? LineItemId { get; set; }
    public long Id { get; set; }
    public string Name { get; set; }
    public decimal Quantity { get; set; }
    public int Price { get; set; }
    public int Amount { get; set; }
    public int? MinPrice { get; set; }
    public int? MaxPrice { get; set; }
    // Dishes нет — конечный уровень
}
```

### Стало

```csharp
// Общая база — поля один раз
public abstract class CalcPositionDto
{
    public Guid? LineItemId { get; set; }
    public long Id { get; set; }
    public string Name { get; set; }
    public decimal Quantity { get; set; }
    public int Price { get; set; }
    public int Amount { get; set; }
    public int? MinPrice { get; set; }
    public int? MaxPrice { get; set; }
}

// Каждый уровень добавляет только своё отличие
public class CalcDishDto : CalcPositionDto
{
    public IEnumerable<CalcIngredientDto> Dishes { get; set; }
    public bool? IsDiscountExcluded { get; set; }
}

public class CalcIngredientDto : CalcPositionDto
{
    public IEnumerable<CalcModificatorDto> Dishes { get; set; }
}

public class CalcModificatorDto : CalcPositionDto
{
    // Конечный уровень — только базовые поля
}
```

Теперь *VatRate* добавляется в одно место и сразу появляется на всех уровнях. Новый уровень иерархии — наследуемся от *CalcPositionDto* и добавляем только то, что отличает его от остальных.

---

## 2.8. Дочерние классы не используют методы и атрибуты родителя

В тестах есть *AlwaysTrueCondition* и *AlwaysFalseCondition* — два класса-заглушки для тестов условий. Наследуют *Condition*, реализуют *IsMatch* одной строчкой. А теперь посмотрим, что они получают от родителя и не используют:

- *ConditionContext* — контекст для доступа к данным. Заглушке не нужен, она ничего не загружает.
- *InitDb* — метод инициализации репозитория. Никогда не вызывается.
- *Description* — текстовое описание для UI. В тестах не читается.
- *IsActive* — флаг активности. Не проверяется.

Из всего родительского контракта используется ровно один абстрактный метод *IsMatch*. Всё остальное — балласт, который наследуется по факту, но мёртв. Это и есть симптом из формулировки: класс наследуется не потому, что нужен родительский функционал, а потому что «надо как-то реализовать абстрактный класс». В таких случаях стоит либо извлечь минимальный интерфейс, который реально нужен потребителю, либо отказаться от наследования вовсе и взять параметризованный класс с лямбдой.

### Было

```csharp
// Два класса-заглушки. Каждый наследует Condition целиком,
// но из всего родителя использует только сигнатуру IsMatch.
internal class AlwaysTrueCondition : Condition
{
    internal override ContextEnum Context =>
        ContextEnum.promotion | ContextEnum.promocode | ContextEnum.discount;

    public override bool IsMatch(Calculation calculation) => true;

    // Унаследованы и не используются:
    //   ConditionContext  — заглушка не обращается к данным
    //   InitDb            — никогда не вызывается
    //   Description       — не читается
    //   IsActive          — не проверяется
}

internal class AlwaysFalseCondition : Condition
{
    internal override ContextEnum Context =>
        ContextEnum.promotion | ContextEnum.promocode | ContextEnum.discount;

    public override bool IsMatch(Calculation calculation) => false;
    // Тот же самый набор неиспользуемых унаследованных членов
}
```

### Стало

Два варианта решения, в зависимости от того, что нужно потребителю.

**Вариант 1 — извлечь интерфейс.** Если потребителю условий на самом деле нужен только *IsMatch*, контракт *Condition* избыточен. Тогда честнее завести узкий интерфейс и принимать его там, где наследование даёт балласт:

```csharp
// Узкий контракт — то, что реально используется
public interface ICondition
{
    bool IsMatch(Calculation calculation);
}

// Боевые условия наследуют Condition (нужны контекст, описание, активность),
// тестовые — реализуют только ICondition без балласта
public abstract class Condition : ICondition
{
    internal IConditionContext ConditionContext { get; private set; }
    public string Description { get; protected set; }
    public bool IsActive { get; protected set; }
    internal void InitDb(IConditionContext ctx) { ConditionContext = ctx; }
    public abstract bool IsMatch(Calculation calculation);
}
```

**Вариант 2 — параметризованная заглушка.** Когда от потомка нужно одно поведение, и оно сводится к функции, наследник вообще не нужен — хватит одного класса, в который поведение передаётся через лямбду:

```csharp
public class StubCondition : ICondition
{
    private readonly Func<Calculation, bool> _match;
    public StubCondition(Func<Calculation, bool> match) => _match = match;
    public bool IsMatch(Calculation calculation) => _match(calculation);
}

// В тестах:
var alwaysTrue  = new StubCondition(_ => true);
var alwaysFalse = new StubCondition(_ => false);
var throwsOnAny = new StubCondition(_ => throw new InvalidOperationException("test"));
```

Оба варианта решают исходную проблему: дочерний класс перестаёт получать в наследство то, чем не пользуется. В первом случае — за счёт более узкого контракта, во втором — за счёт отказа от наследования в пользу композиции.

---

## 3.1. Одна модификация требует внесения изменений в несколько классов

В системе три инструмента лояльности: *Promotion*, *Promocode*, *Discount*. У каждого две команды: создать и отредактировать. Итого шесть классов запросов. Открываешь их по очереди и видишь одно и то же: *Name*, *Description*, *IsActive*, *IsCombined*, *Start*, *End*, *Limit*, *Conditions*, *Reward*. Шесть файлов, один набор полей.

Пока никто ничего не меняет — жить можно. Проблема приходит, когда появляется новое бизнес-требование. Допустим, нужно добавить *Tags* для категоризации инструментов. Добавляем в первый запрос — всё, первый готов. Потом вспоминаем про второй. Потом про третий. Потом про хэндлеры. Потом про доменные методы. Потом про контроллеры. Одно решение — минимум девять правок, и каждая это шанс что-то забыть или сделать по-другому.

### Было

```csharp
// CreateEmptyPromotionRequest — 9 параметров
public class CreateEmptyPromotionRequest : IRequest<PromotionDto>
{
    public CreateEmptyPromotionRequest(string name, string description, bool isCombined, bool isActive,
        DateTimeOffset? start, DateTimeOffset? end, int? limit,
        List<(string name, dynamic options)> conditions,
        (string name, dynamic options) reward) { ... }

    public string Name { get; }
    public string Description { get; }
    public bool IsCombined { get; }
    public bool IsActive { get; }
    public int? Limit { get; }
    public DateTimeOffset? Start { get; }
    public DateTimeOffset? End { get; }
    public List<(string name, dynamic options)> Conditions { get; }
    public (string name, dynamic options) Reward { get; }
}

// EditPromocodeRequest — те же поля, другой класс
public class EditPromocodeRequest : IRequest<PromocodeDto>
{
    public EditPromocodeRequest(Guid id, string name, string description, bool isActive, bool isCombined,
        DateTimeOffset? start, DateTimeOffset? end, int? limit,
        List<(string name, dynamic options)> conditions,
        (string name, dynamic options) reward) { ... }
    // ... те же 9 свойств
}

// CreateDiscountRequest — снова те же поля, другой класс
public class CreateDiscountRequest : IRequest<DiscountDto>
{
    public CreateDiscountRequest(string name, string description, bool isActive,
        List<(string name, dynamic options)> conditions,
        (string name, dynamic options) reward) { ... }
    // ... те же свойства (только без Start/End/Limit — Discount не поддерживает)
}
```

### Стало

```csharp
// Общие настройки инструмента лояльности — одно место.
// Добавить Tags — одна строка здесь, и она автоматически появится во всех шести запросах.
public class BenefitSettings
{
    public string Name { get; init; }
    public string Description { get; init; }
    public bool IsActive { get; init; }
    public bool IsCombined { get; init; }
    public DateTimeOffset? Start { get; init; }
    public DateTimeOffset? End { get; init; }
    public int? Limit { get; init; }
    public List<(string name, dynamic options)> Conditions { get; init; }
    public (string name, dynamic options) Reward { get; init; }
}

// Каждый запрос — только то, что отличает его от остальных
public class CreatePromotionRequest : IRequest<PromotionDto>
{
    public CreatePromotionRequest(BenefitSettings settings) { Settings = settings; }
    public BenefitSettings Settings { get; }
}

public class EditPromotionRequest : IRequest<PromotionDto>
{
    public EditPromotionRequest(Guid id, BenefitSettings settings) { Id = id; Settings = settings; }
    public Guid Id { get; }
    public BenefitSettings Settings { get; }
}

// CreatePromocodeRequest, EditPromocodeRequest, CreateDiscountRequest, EditDiscountRequest
// — та же структура, разный тип результата
```

Добавление *Tags* — одна строка в *BenefitSettings*, и всё. Маппинг из HTTP-тела в контроллере пишется один раз для всех инструментов. Забыть добавить поле в один из шести классов больше невозможно.

---

## 3.2. Использование сложных паттернов там, где можно проще

В системе есть три места, которые умеют возвращать список доступных условий: хэндлер для скидок, для акций и для промокодов. На первый взгляд кажется нормальным — каждый модуль знает свои условия. Но открываешь все три и видишь буквально одинаковый код: *FindImplementsExtension.Find\<Condition\>*, фильтр по *ContextEnum*, маппинг в *ListConditionDto*. Единственное отличие — одна строка с флагом контекста. У *GetRewardsRequest* та же история, ещё три копии.

Цена этого удовольствия: захотел поменять формат *ListConditionDto* — три файла. Нашёл баг в логике фильтрации — исправляй в трёх местах и молись, чтобы не пропустить четвёртый.

### Было

```csharp
// Discounts.Queries.RequestHandler
public async ValueTask<IEnumerable<ListConditionDto>> Handle(
    GetConditionsRequest request, CancellationToken ct)
{
    return FindImplementsExtension.Find<Condition>("Loyalty.Domain.Conditions")
        .Where(x => x.Context.HasFlag(ContextEnum.discount))  // ← единственное отличие
        .Select(x => new ListConditionDto
        {
            Context = x.Context.ToString(),
            Name = x.GetType().Name,
            Description = x.Description,
            IsActive = x.IsActive,
            Params = x
        });
}

// Promotions.Queries.RequestHandler — идентично, только флаг другой
public async ValueTask<IEnumerable<ListConditionDto>> Handle(
    GetConditionsRequest request, CancellationToken ct)
{
    return FindImplementsExtension.Find<Condition>("Loyalty.Domain.Conditions")
        .Where(x => x.Context.HasFlag(ContextEnum.promotion))  // ← единственное отличие
        .Select(x => new ListConditionDto { ... });             // остальное — копипаста
}

// Promocodes.RequestHandler — то же самое, третий раз
public async ValueTask<IEnumerable<ListConditionDto>> Handle(
    GetConditionsRequest request, CancellationToken ct)
{
    return FindImplementsExtension.Find<Condition>("Loyalty.Domain.Conditions")
        .Where(x => x.Context.HasFlag(ContextEnum.promocode))  // ← единственное отличие
        .Select(x => new ListConditionDto { ... });
}
```

### Стало

```csharp
// Один обобщённый сервис на всё — условия и награды отличаются только типом базового класса
// и пространством имён. Дженерик закрывает оба измерения, копий формата DTO больше нет.
public class BenefitCatalogService
{
    public IEnumerable<ListItemDto> GetItems<T>(string @namespace, ContextEnum context)
        where T : IBenefitItem
    {
        return FindImplementsExtension.Find<T>(@namespace)
            .Where(x => x.Context.HasFlag(context))
            .Select(x => new ListItemDto
            {
                Context = x.Context.ToString(),
                Name = x.GetType().Name,
                Description = x.Description,
                IsActive = x.IsActive,
                Params = x
            });
    }
}

// Общий контракт для условий и наград — то, что нужно сервису для маппинга
public interface IBenefitItem
{
    ContextEnum Context { get; }
    string Description { get; }
    bool IsActive { get; }
}

// Все шесть хэндлеров (3 контекста × 2 типа) — одна строка делегирования
public ValueTask<IEnumerable<ListItemDto>> Handle(GetConditionsRequest request, CancellationToken ct)
    => ValueTask.FromResult(_catalog.GetItems<Condition>("Loyalty.Domain.Conditions", ContextEnum.discount));

public ValueTask<IEnumerable<ListItemDto>> Handle(GetRewardsRequest request, CancellationToken ct)
    => ValueTask.FromResult(_catalog.GetItems<Reward>("Loyalty.Domain.Rewards", ContextEnum.discount));
```

Изменить формат *ListItemDto* — одно место. Баг в фильтрации — правишь один раз, покрываешь все шесть хэндлеров. И главное: при попытке сделать «как было» (отдельный метод *GetConditions* и отдельный *GetRewards*) внутри сервиса осталась бы та же копипаста, просто перенесённая на этаж ниже. Дженерик честнее — он видит, что условия и награды разные только по типу, и ровно это и выражает.

---

## Общая рефлексия

Уровень классов — это где проблемы перестают быть косметическими. На уровне строк всё поправимо за несколько минут. Здесь цена выше: класс, который оброс лишними обязанностями, не рефакторится за вечер. Особенно если на него завязаны тесты, другие классы и контроллеры.

Что зацепило больше всего в этих примерах: большинство проблем появились не от лени и не от незнания. *CalcDishDto*/*CalcIngredientDto*/*CalcModificatorDto* создавались в разное время, каждый раз казалось что «вот этот немного другой». Три хэндлера с одинаковым кодом — тоже не копипаст от безделья, просто каждый модуль добавлялся отдельно. Эти вещи незаметны в моменте и очень заметны когда нужно что-то менять.

Простой вопрос, который помогает ловить такие вещи раньше: если завтра добавится похожий инструмент — сколько мест придётся менять? Если больше одного — скорее всего, общая концепция пока не выделена.