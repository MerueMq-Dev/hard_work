# Отчёт: Пять страховок «от дурака», которые я хочу убрать дизайном

## Введение

Главное, что я понял из материала: если в коде стоят длинные цепочки *if-null*, *if-empty*, *ToLower-сравнения* и прочая защита от мусора — обычно дело не в том, что нужны проверки получше. Дело в том, что код вообще позволяет передать в это место такие значения.

В материале это сравнивается с инструкцией к плееру: «не задавайте отрицательную громкость, иначе зависнет». То есть проблема не в том, что инструкция плохо написана. Проблема в том, что плеер вообще допускает отрицательную громкость. Правильный дизайн — это когда такое значение в принципе нельзя ввести.

Я прошёлся по проекту и нашёл пять разных мест, где у меня такая страховка стоит. Не доменные правила (типа «нельзя закрыть расчёт без гостя» — это нормальное бизнес-условие), а именно ту самую защиту от значений, которых в этом месте быть не должно было. И для каждого попробовал придумать, как переместить страховку выше или вообще убрать.

---

## Случай 1. *ExpeditionType.IsMatch* — *null*, пустая строка и сравнение без регистра

### Что у меня сейчас

```csharp
public class ExpeditionType : Condition
{
    // варианты: pickup | delivery
    public string Value { get; set; }

    public override bool IsMatch(Calculation calculation)
    {
        if (Value == null)
            return false;

        if (calculation.Order?.ExpeditionType == null)
            return false;

        return string.Equals(
            calculation.Order.ExpeditionType, Value, StringComparison.InvariantCultureIgnoreCase);
    }
}
```

И в тестах у меня прямо явно перечислены кейсы: *DELIVERY/delivery*, *null*, *""*. Я их сам туда положил, потому что знал — такое реально может прийти.

### Почему это страховка от мусора

Тип данных — *string Value* — позволяет туда положить что угодно: *"pickup"*, *"delivery"*, *"DELIVERY"*, *"deliveri"*, *null*, *""*, *"42"*. А у бизнеса вообще-то всего два значения: pickup и delivery. То есть тип в три раза шире, чем реальность, и я плачу за это проверками в каждом месте использования.

Прямо как с плеером. Я в коде по сути пишу: «осторожно, не передавайте сюда *null* или строку с большой буквы».

### Стало

```csharp
public enum ExpeditionTypeValue { Pickup, Delivery }

public class ExpeditionType : Condition
{
    public ExpeditionTypeValue Value { get; set; }

    public override bool IsMatch(Calculation calculation)
    {
        // Никаких проверок на null и приведений к нижнему регистру.
        // Других значений в типе просто нет.
        return calculation.Order?.ExpeditionType == Value;
    }
}
```

*calculation.Order.ExpeditionType* я тоже делаю *ExpeditionTypeValue*. Это изменение, которое расползётся по нескольким файлам, но именно оно убивает страховку в корне. Парсинг строки из внешнего API я уношу в одно место — в маппер на входе. Внутри системы строки больше не будет.

### Что я удалил и что улучшил

Удалил: проверку *Value == null*, проверку *order?.ExpeditionType == null*, сравнение *InvariantCultureIgnoreCase*. Тестовые кейсы с *""*, *null* и разным регистром стали невозможны в принципе. Из теории на десять *InlineData* остаётся два сценария — совпало или нет. Страховка переехала на вход системы, а внутри её нет — потому что охранять стало нечего.

---

## Случай 2. *FindImplementsExtension* — проверка наличия конструктора в рантайме

### Что у меня сейчас

```csharp
internal static class FindImplementsExtension
{
    public static IEnumerable<T> Find<T>(string namespaceValue)
    {
        // ... ищем все наследники T в указанном namespace ...

        foreach (var type in implementationTypes)
        {
            var constructor = type.GetConstructor(Type.EmptyTypes);
            if (constructor == null)
            {
                throw new InvalidOperationException(
                    $"Type {type.FullName} does not have a parameterless constructor");
            }

            yield return (T)Activator.CreateInstance(type);
        }
    }
}
```

Я использую это в трёх местах — в *GetConditionsRequest* для Discounts, Promotions и Promocodes — чтобы выдать список доступных условий.

### Почему это страховка от мусора

Метод говорит: «дайте мне любой наследник *Condition* из такого-то namespace, я создам его через *Activator.CreateInstance*». Если у наследника не окажется пустого конструктора — кидаю *InvalidOperationException* в рантайме.

Это я компенсирую дыру в C#: язык не умеет выразить «у всех наследников должен быть конструктор без параметров». Поэтому проверка живёт там, где ошибка вылезает — в момент перечисления. И, что неприятно, ошибка вылезет не у того, кто добавил новое условие без конструктора, а у того, кто откроет страницу со списком условий и получит непонятный 500.

### Стало

Самое простое — явный реестр условий, без рефлексии вообще:

```csharp
// Условия регистрируются явно при старте приложения, а не находятся рефлексией
public static class ConditionCatalog
{
    private static readonly Dictionary<string, Func<Condition>> _factories = new();

    public static void Register<TCondition>(Func<TCondition> factory) where TCondition : Condition
        => _factories[typeof(TCondition).Name] = () => factory();

    public static IReadOnlyDictionary<string, Func<Condition>> Items => _factories;
}

// Где-то при старте:
ConditionCatalog.Register(() => new ExpeditionType());
ConditionCatalog.Register(() => new GuestBirthday());
ConditionCatalog.Register(() => new ForFirstOrder());
// ... остальные
```

Чтобы добавить новое условие, теперь надо явно зарегистрировать его — и в этот момент компилятор сам проверит, что фабрика возвращает совместимый тип. Забыть про конструктор невозможно: я сам решаю, как создаётся объект, и пишу это руками.

### Что я удалил и что улучшил

Удалил: проверку *if constructor == null throw InvalidOperationException* и неявное требование «все наследники должны иметь parameterless ctor». Заодно убрал *Activator.CreateInstance* и всю возню с рефлексией. Страховка переехала из рантайма в момент сборки: если я забуду зарегистрировать новое условие — это станет понятно сразу при добавлении, а не «когда-нибудь, когда кто-то откроет нужную страницу».

---

## Случай 3. Проверка *options.Value?.RabbitQueue == null* в трёх конструкторах подряд

### Что у меня сейчас

В нескольких сервисах, которые работают с RabbitMQ, в конструкторе живёт одна и та же страховка:

```csharp
// Loyalty.Infrastructure.Licenses.Imports.RequestHandler
public RequestHandler(
    /* ... другие зависимости ... */,
    IOptions<LoyaltyMenuServiceOptions> options)
{
    /* ... присваиваем зависимости ... */

    if (options.Value?.RabbitQueue == null)
        throw new ArgumentNullException(nameof(options), "LoyaltyMenuService__RabbitMQ is null");

    _menuServiceQueue = options.Value.RabbitQueue;
}

// Loyalty.Infrastructure.Sagas.Menu.MenuSagaService
public MenuSagaService(
    /* ... другие зависимости ... */,
    IOptions<LoyaltyMenuServiceOptions> options)
{
    /* ... */

    if (options.Value?.RabbitQueue == null)
        throw new ArgumentNullException(nameof(options), "LoyaltyMenuService__RabbitMQ is null");

    _menuServiceQueue = options.Value.RabbitQueue;
}

// Loyalty.Infrastructure.Sagas.Menu.RequestHandler — третий раз тот же блок
if (options.Value?.RabbitQueue == null)
    throw new ArgumentNullException(nameof(options), "LoyaltyMenuService__RabbitMQ is null");

_menuServiceQueue = options.Value.RabbitQueue;
```

Три места, один и тот же блок из четырёх строк со строкой-литералом сообщения.

### Почему это страховка от мусора

*LoyaltyMenuServiceOptions* — это просто класс с полем *string? RabbitQueue { get; set; }*, в который ASP.NET складывает значения из конфигурации. Сам тип никак не выражает «это поле обязательное». Поэтому в каждом конструкторе, который этот options использует, я вынужден дописать одну и ту же проверку. Если забуду — вместо понятной ошибки на старте получу *NullReferenceException* где-то в недрах, в момент когда сага попытается отправить команду в очередь.

И самое неприятное — это дублирование одного и того же блока. Если я завтра решу, что сообщение об ошибке должно быть другим, или что нужно ещё проверить формат URI очереди — мне нужно править три места. И ничто не мешает кому-то добавить четвёртый сервис и забыть скопировать проверку.

### Стало

Делаю поле обязательным на уровне самого options-класса и валидирую один раз при старте:

```csharp
public class LoyaltyMenuServiceOptions
{
    [Required(AllowEmptyStrings = false)]
    public string RabbitQueue { get; init; } = null!;
}

// Один раз при сборке хоста:
builder.Services
    .AddOptions<LoyaltyMenuServiceOptions>()
    .Bind(builder.Configuration.GetSection("LoyaltyMenuService"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

И во всех трёх конструкторах:

```csharp
public RequestHandler(
    /* ... */,
    IOptions<LoyaltyMenuServiceOptions> options)
{
    /* ... */
    _menuServiceQueue = options.Value.RabbitQueue;   // никаких проверок, просто используем
}
```

### Что я удалил и что улучшил

Удалил три копии блока *if options.Value?.RabbitQueue == null throw ArgumentNullException(...)* со строкой-литералом «LoyaltyMenuService__RabbitMQ is null». Сам по себе блок мелкий, но он был в трёх местах — и это уже не просто страховка, а ещё и дублирование.

Страховка переехала с уровня «каждый конструктор, который читает options, дописывает свой *if*» на уровень «опции валидируются один раз при старте». Если очередь не сконфигурирована — приложение просто не запустится, и я узнаю об этом сразу, а не через час, когда сага попытается что-то отправить. И добавить четвёртый сервис, использующий тот же options, теперь безопасно — никаких ритуальных *if*-ов копировать не надо.

---

## Случай 4. *ConvertOrDefault* / *ConvertOrNull* — парсинг чисел из строк

### Что у меня сейчас

```csharp
public static class StringConversionExtensions
{
    public static T? ConvertOrNull<T>(this string value) where T : struct
    {
        if (string.IsNullOrEmpty(value))
            return null;
        // ... TypeConverter, ConvertFromString
    }

    public static T ConvertOrDefault<T>(this string value) where T : struct
    {
        if (string.IsNullOrEmpty(value))
            return default(T);
        // ... TypeConverter, ConvertFromString
    }
}
```

Я этим пользуюсь, чтобы разобрать числовые поля *MenuProduct* — *Price*, *Protein*, *Fat*, *Carbohydrates*, *Kcal* — которые у меня объявлены как *string*.

### Почему это страховка от мусора

Это прямо как с плеером. *string Price* я объявил «потому что внешнее API так присылает». Но внутри системы каждый, кто хочет с этой ценой что-то сделать, должен дёрнуть *ConvertOrDefault* — иначе нельзя ни сравнить с нулём, ни сложить.

Получается, страховка от того, что «строка может быть пустой или непарсимой», размазана по всему коду через *ConvertOrDefault*. И если я где-то забуду её вызвать — получу либо *null*, либо рантайм-исключение глубоко в бизнес-логике.

### Стало

Парсинг происходит один раз — на входе, в маппере. Дальше по системе ездит уже типизированный *Product*:

```csharp
// MenuProduct остаётся «зеркалом» внешнего API — пусть будет таким, как пришло
public class MenuProduct
{
    public string Id { get; set; }
    public string Price { get; set; }
    public string Protein { get; set; }
    // ...
}

// Маппер — единственное место, где разбираем строки
public static class MenuProductMapper
{
    public static Product ToDomain(MenuProduct dto, long corporationId, long restaurantId)
    {
        return new Product(
            corporationId, restaurantId,
            id:            long.Parse(dto.Id),
            name:          dto.Name,
            price:         ParseDecimalOrZero(dto.Price),
            protein:       ParseFloatOrZero(dto.Protein),
            fat:           ParseFloatOrZero(dto.Fat),
            carbohydrates: ParseFloatOrZero(dto.Carbohydrates),
            kcal:          ParseFloatOrZero(dto.Kcal));
    }

    private static decimal ParseDecimalOrZero(string s) =>
        decimal.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var v) ? v : 0m;

    private static float ParseFloatOrZero(string s) =>
        float.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var v) ? v : 0f;
}

// Везде внутри системы — никаких ConvertOrDefault.
// Работаем с decimal Price, не со string Price:
if (product.Price > 0) { /* обычное сравнение чисел */ }
```

### Что я удалил и что улучшил

Удалил все вызовы *.ConvertOrDefault<decimal>()* и *.ConvertOrNull<int>()* в местах использования. Сам helper *StringConversionExtensions* теперь не нужен — можно выкинуть.

Граница между «как пришло снаружи» и «как у меня внутри» теперь явная. Парсинг происходит один раз в маппере, дальше по коду едут честные *decimal* и *float*. Случайно сравнить *"abc"* с числом и получить исключение в глубине бизнес-логики стало невозможно — таких строк в системе просто больше нет.

---

## Случай 5. *AcceptWithdrawBonuses* / *DeclineWithdrawBonuses* — три «базовых проверки» в начале метода

### Что у меня сейчас

```csharp
public bool AcceptWithdrawBonuses(int amount)
{
    // базовые проверки
    if (Status.IsCompleted())
        return false;

    if (Receipts.Any())
        return false;

    // проверка бонусной системы
    if (WithdrawBonusStatus != WithdrawStatusEnum.requested)
        return false;

    WithdrawBonusStatus = WithdrawStatusEnum.yes;
    WithdrawBonusActualAmount = amount;
    ConcurrencyToken = Guid.NewGuid();
    UpVersion();
    return true;
}

public bool DeclineWithdrawBonuses()
{
    // те же три проверки, тот же шаблон, отличается только присваивание в конце
    if (Status.IsCompleted()) return false;
    if (Receipts.Any())       return false;
    if (WithdrawBonusStatus != WithdrawStatusEnum.requested) return false;

    WithdrawBonusStatus = WithdrawStatusEnum.no;
    ConcurrencyToken = Guid.NewGuid();
    UpVersion();
    return true;
}
```

### Почему это страховка от мусора

Тут нужно сразу оговориться. Сами по себе бизнес-правила — это не страховка от дурака. Расчёт действительно не должен принимать решения по бонусам, если он закрыт или если уже выписаны чеки. Так и должно быть.

Что выдаёт здесь именно страховку — это повтор трёх одинаковых проверок в двух методах и форма «булева проверка → *return false*». Метод тихо ничего не делает и возвращает *false* в трёх разных ситуациях — а вызывающий код не различает, что именно пошло не так. И эти проверки в начале нужны мне не потому что они здесь полезны, а потому что у расчёта слишком развязанное состояние: технически у меня может быть расчёт с *Status == active*, *Receipts.Any() == true* и *WithdrawBonusStatus == requested* одновременно. Бизнес говорит — так не бывает. Тип данных — спокойно бывает.

То есть три проверки в начале — это симптом того, что комбинация *Status × hasReceipts × WithdrawBonusStatus* шире, чем реальные допустимые состояния.

### Стало

Состояние бонусной операции стягиваю в одно поле, в котором заранее зашиты все допустимые варианты:

```csharp
// Состояние бонусной операции внутри расчёта — одно поле, не три.
// Несовместимые комбинации просто нельзя выразить.
public abstract record WithdrawBonusState
{
    public sealed record NotRequested : WithdrawBonusState;
    public sealed record Requested(int Amount) : WithdrawBonusState;
    public sealed record Accepted(int ActualAmount) : WithdrawBonusState;
    public sealed record Declined : WithdrawBonusState;
}

public class Calculation
{
    public WithdrawBonusState Withdraw { get; private set; } = new WithdrawBonusState.NotRequested();

    // Принять решение можно только если бонусы вообще запрошены.
    // Других вариантов нет.
    public bool AcceptWithdrawBonuses(int amount)
    {
        if (Withdraw is not WithdrawBonusState.Requested r)
            return false;

        Withdraw = new WithdrawBonusState.Accepted(amount);
        ConcurrencyToken = Guid.NewGuid();
        UpVersion();
        return true;
    }
    // Decline — симметрично, через WithdrawBonusState.Declined
}
```

А проверки на *Status.IsCompleted()* и *Receipts.Any()* я уношу выше — туда, где саги/координатор и так знают, в каком состоянии расчёт, и решают, можно ли вообще запрашивать бонусы. То есть состояние *Requested* просто не появится у закрытого расчёта или у расчёта с уже выставленным чеком.

### Что я удалил и что улучшил

Удалил дублирующиеся «базовые проверки» из обоих методов. Заодно убрал саму возможность вызвать *Accept*/*Decline* в неправильный момент — теперь это не «вызов ничего не сделал и вернул *false*», а ситуация, которую просто нельзя описать. Связку из *WithdrawBonusStatus + WithdrawBonusActualAmount* стянул в одно типизированное поле — нельзя случайно получить комбинацию «статус no, а сумма 500».

Это уже даже не «защита» — это просто разные состояния выражены так, что несовместимые комбинации в коде записать нельзя.

---

## Что у меня получилось в итоге

Когда я смотрю на все пять случаев вместе, виден один и тот же узор. Каждый раз источник проблемы был один: тип данных у меня шире, чем реальные допустимые значения.

- *string Value* — шире, чем *{pickup, delivery}*.
- «Любой наследник *Condition*» — шире, чем «наследник, который умеет создаваться без параметров».
- Класс options c полем *string? RabbitQueue* — шире, чем «это поле обязано быть заполнено».
- *string Price* — шире, чем *decimal*.
- Три независимых булевых поля состояния — шире, чем четыре реальных состояния бонусной операции.

И везде, где у меня тип шире реальности, в коде появлялись страховки. Не потому что я ленивый или невнимательный — просто потому что в этих местах действительно могло оказаться что угодно. И единственный способ убрать страховку честно — это сузить тип до того, что у меня реально бывает. Тогда защищать становится нечего.

Главное, что я хочу запомнить: когда я в очередной раз тянусь дописать *if-проверку* в начале метода, это сигнал. Не «надо добавить ещё одну проверку понадёжнее», а «где-то выше по коду тип описан с запасом, и за этот запас я плачу каждым местом использования». Лучше потратить полчаса там, чем сорок раз дописывать одинаковые *if* потом.