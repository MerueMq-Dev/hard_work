# Антипаттерн «Интерфейс компактнее реализации»

## С чего начать

Когда я первый раз читал про призрачное состояние и погрешность спецификации, казалось — ну, теория, бывает. Потом открыл наш код и нашёл всё это буквально за полчаса. Причём в местах, где мы ходим каждый день и считаем их «нормальными». Это и есть самое неприятное в таких вещах: они не выглядят как проблема, пока не смотришь специально.

Ниже — девять конкретных мест из проекта. Три категории, по три примера в каждой.

---

## Часть 1. Призрачное состояние

Призрачное состояние — это когда переменная живёт в нескольких смысловых режимах, но тип этого не отражает. Компилятор доволен, тесты зелёные, а читатель сидит и думает: «подождите, а что тут вообще происходит?»

### 1.1. *calculation = null* — переменная с тремя жизнями

Вот этот кусок есть сразу в двух хэндлерах — *PreCalculations* и *Calculations*. Слово в слово:

```csharp
Calculation calculation = null;

if (!string.IsNullOrEmpty(request.Token))
    calculation = await _factory.CreateByToken(...);

if (!string.IsNullOrEmpty(request.Phone))
    calculation = await _factory.CreateByPhone(...);

if (calculation == null)
    throw new ApiException(Enums.ApiStatus.FORBIDDEN_WITH_MESSAGE, "Антифрод: превышен лимит заказов");
```

На первый взгляд — ничего страшного. Но давайте честно: переменная *calculation* здесь живёт в трёх состояниях, ни одно из которых не описано явно. Сначала она «ещё неизвестно». Потом — «определена по токену». Потом — и это самое интересное — она может быть **молча перезаписана** по телефону, если переданы оба значения.

Никакого приоритета, никакого предупреждения. Второй *if* просто затирает первый. И это не теоретическая проблема — стоит кому-то передать и токен, и телефон одновременно (а такое случается в интеграциях), расчёт по токену исчезнет без следа.

И ещё одна деталь: когда *calculation* всё-таки остаётся *null*, мы кидаем ошибку про антифрод. Антифрод тут совершенно ни при чём — просто никто не передал идентификатор гостя. Это как написать «нет интернета», когда на самом деле кабель не подключён.

```csharp
// Так намерение читается без расследования
Calculation calculation;

if (!string.IsNullOrEmpty(request.Token))
    calculation = await _factory.CreateByToken(...);
else if (!string.IsNullOrEmpty(request.Phone))
    calculation = await _factory.CreateByPhone(...);
else
    throw new ApiException(Enums.ApiStatus.ERR_PARSE_REQUEST,
        "Необходимо передать токен гостя или номер телефона");
```

Три строчки разницы — и приоритет токена стал явным, ошибка стала честной, а призрак исчез.

---

### 1.2. *ElapsedTime* — геттер, который знает секрет

```csharp
public TimeSpan? ElapsedTime
{
    get
    {
        if (Updated == null)
            return DateTime.UtcNow - Created;

        if (State == "Final" || State == "completed")
            return Updated.Value - Created;

        return DateTime.UtcNow - Created;
    }
}
```

С первого взгляда — просто вычисление времени. Но внутри спрятано кое-что интересное: этот геттер знает, что *"Final"* и *"completed"* означают «процесс закончен». Это знание нигде больше не зафиксировано — только здесь, в строковом сравнении.

Откуда взялось *"Final"*? Это состояние из *UpdateState* (сага обновления меню). А *"completed"* — из *ImportSagaState* (сага импорта). Две разные системы, два разных enum'а, и оба их финальных состояния собраны в одну строчку через *||*. Добавится третья сага с финальным состоянием *"done"* — и *ElapsedTime* начнёт считать время «до сих пор» для уже завершённых процессов. Молча. Без единого теста, который это поймает.

```csharp
// Теперь понятие «завершённость» живёт в одном месте
private bool IsCompleted => State == UpdateState.completed.ToString()
                         || State == ImportSagaState.completed.ToString();

public TimeSpan? ElapsedTime => IsCompleted
    ? Updated!.Value - Created
    : DateTime.UtcNow - Created;
```

*IsCompleted* — это не просто рефакторинг. Это имя для концепта, который раньше существовал только неявно.

---

### 1.3. *ConditionContext* — объект, который «не готов»

```csharp
public abstract class Condition
{
    internal void InitDb(IConditionContext context)
    {
        ConditionContext = context;
    }

    protected IConditionContext ConditionContext { get; private set; }
}
```

Вот классика жанра. Объект создан — но использовать его нельзя, пока не вызовешь *InitDb*. Это именно то, что называют призрачным состоянием: между *new GuestSegments()* и *condition.InitDb(...)* объект существует, но он сломан. Вызови *IsMatch* в этот момент — получишь *NullReferenceException* прямо из *ConditionContext.GetRepository\<T\>()*.

Ни тип, ни компилятор об этом не предупредят. Контракт нигде не написан. Узнаешь только в рантайме.

Самое показательное: *InitDb* существует потому, что условия десериализуются из JSON через рефлексию — конструктор с параметрами туда не передашь. Это реальное техническое ограничение, и оно понятно. Но из этого ограничения выросло состояние «объект создан, но не инициализирован» — и оно теперь является частью публичного (ну, *internal*) контракта класса. Это не намеренное решение, это просто так получилось.

---

## Часть 2. Погрешность/неточность спецификации

Если призрачное состояние — это про «что», то неточность — про «насколько точно». Реализация говорит одно конкретное число, а спецификация должна говорить диапазон или правило. Когда допущение о конкретном значении ломается — всё, что на него опиралось, ломается вместе с ним.

### 2.1. *OrderAmount* — магическое *× 100* без объяснений

```csharp
public class OrderAmount : Condition
{
    public decimal MinOrderAmount { get; set; }

    public override bool IsMatch(Calculation calculation)
    {
        return calculation.TotalAmount >= MinOrderAmount * 100; // переводим в копейки
    }
}
```

Комментарий «переводим в копейки» — это честно. Но честность комментария не делает контракт явным. Имя *MinOrderAmount* не говорит ни слова о единицах. Кто-то читает это поле — и не знает: 500 — это 500 рублей или 500 копеек? Лезет смотреть в базу, потом в тесты, потом находит комментарий и успокаивается. Через полгода приходит другой человек и проходит тот же путь.

А ещё рядом в коде уже есть *ToRubles()* — метод, который конвертирует копейки в рубли для отображения. Конвертации идут в обе стороны, но только одна из них документирована в типе. Это и есть неточность: спецификация метода молчит о том, что является фундаментальным для его работы.

```csharp
/// <summary>
/// Условие: минимальная сумма заказа.
/// MinOrderAmountRubles задаётся в рублях (например, 500 = 500 ₽).
/// Сравнение производится с TotalAmount, который хранится в копейках.
/// Если единицы хранения TotalAmount изменятся — конвертацию пересмотреть здесь.
/// </summary>
public class OrderAmount : Condition
{
    public decimal MinOrderAmountRubles { get; set; }

    public override bool IsMatch(Calculation calculation)
    {
        var minAmountKopecks = MinOrderAmountRubles * 100;
        return calculation.TotalAmount >= minAmountKopecks;
    }
}
```

Переименование поля + XML-doc — и следующий разработчик не будет гадать.

---

### 2.2. *CheckInGuestRequest* — успех или провал, третьего не дано

```csharp
public class CheckInGuestRequest : IRequest<(bool success, string orderNumber)>
```

```csharp
// Хэндлер внутри:
try
{
    calculation = await _factory.GetAndCheckin(...);
    await repository.Save(calculation);
    _unitOfWork.Commit();
}
catch (Exception ex)
{
    _unitOfWork.Rollback();
    _logger.LogError(ex, ex.Message);
    return (false, null);  // ← всё в одну кучу
}

return (true, calculation.OrderNumber);
```

Вот тут неточность бьёт в полную силу. Спецификация возврата — *(bool, string)* — подразумевает мир с двумя состояниями: «всё хорошо» и «что-то пошло не так». Но реальность богаче: не найден расчёт по токену, антифрод заблокировал, база упала в середине транзакции, коммит не прошёл — всё это возвращается одним и тем же *(false, null)*. Вызывающий код видит только «не получилось» и не может ни обработать ситуацию по-разному, ни даже нормально залогировать.

Это неточность в спецификации: она описывает интерфейс уже, чем нужно для реального использования.

```csharp
/// <summary>
/// Результат привязки гостя к расчёту.
/// При Success = true: OrderNumber содержит номер заказа, Reason = null.
/// При Success = false: OrderNumber = null, Reason описывает причину сбоя.
/// NotFound — расчёт не найден по токену.
/// StorageError — ошибка при сохранении или коммите транзакции.
/// AntifraudBlocked — антифрод не пропустил операцию.
/// </summary>
public record CheckInResult(bool Success, string? OrderNumber, CheckInFailReason? Reason = null);

public enum CheckInFailReason { NotFound, StorageError, AntifraudBlocked }
```

Теперь вызывающий код может реагировать осмысленно: показать одно сообщение при *NotFound* и совсем другое при *StorageError*.

---

### 2.3. *GuestBirthday.IsMatch* — день рождения, который никогда не наступит

```csharp
public override bool IsMatch(Calculation calculation)
{
    var birthdayDate = calculation.Guest.Birthday;
    var currentDate = DateOnly.FromDateTime(calculation.CreateDate.Date);

    var startDate = currentDate.AddDays(-(DaysBefore ?? 5));
    var endDate = currentDate.AddDays(DaysAfter ?? 5);

    return birthdayDate >= startDate && birthdayDate <= endDate;
}
```

Это мой любимый пример в этом разделе, потому что ошибка абсолютно неочевидна — пока не запустишь в продакшене и не получишь ноль срабатываний на акцию «ко дню рождения».

*birthdayDate* — это дата типа *1990-03-15*. *startDate* и *endDate* — это даты вроде *2025-03-10* и *2025-03-20*. Сравниваем 1990 год с 2025-м — и, конечно, условие никогда не выполнится. Технически код правильный. Семантически — сломан с первого дня.

Спецификация метода нигде не сказала явно: «мы сравниваем только день и месяц, год игнорируется». Это допущение жило в голове автора, но не попало в код. Результат — молчащий баг: акции ко дню рождения не работают, никаких исключений нет, никаких красных флагов нет.

```csharp
/// <summary>
/// Условие: день рождения гостя попадает в окно вокруг текущей даты.
/// ГОД РОЖДЕНИЯ ИГНОРИРУЕТСЯ — сравниваются только день и месяц.
/// DaysBefore (по умолчанию 5): за сколько дней до ДР условие считается выполненным.
/// DaysAfter (по умолчанию 5): сколько дней после ДР условие считается выполненным.
/// Переход через конец/начало года (31 дек → 1 янв) обрабатывается корректно.
/// </summary>
public override bool IsMatch(Calculation calculation)
{
    if (calculation?.Guest?.Birthday == null)
        return false;

    var birthday = calculation.Guest.Birthday.Value;
    var today = DateOnly.FromDateTime(calculation.CreateDate.Date);

    // Переносим ДР в текущий год — именно здесь год рождения перестаёт иметь значение
    var birthdayThisYear = new DateOnly(today.Year, birthday.Month, birthday.Day);
    var start = birthdayThisYear.AddDays(-(DaysBefore ?? 5));
    var end = birthdayThisYear.AddDays(DaysAfter ?? 5);

    // Граничный случай: ДР в декабре, сегодня начало января — смотрим прошлый год
    if (start.Year < today.Year)
    {
        var birthdayLastYear = new DateOnly(today.Year - 1, birthday.Month, birthday.Day);
        var startLastYear = birthdayLastYear.AddDays(-(DaysBefore ?? 5));
        var endLastYear = birthdayLastYear.AddDays(DaysAfter ?? 5);
        if (today >= startLastYear && today <= endLastYear)
            return true;
    }

    return today >= start && today <= end;
}
```

---

## Часть 3. Когда интерфейс не должен быть проще реализации

Это самая неочевидная часть. Обычно нас учат: «делай интерфейс простым». Но есть случаи, когда упрощение интерфейса — это не абстракция, а ложь. Когда за *string* или кортежем прячется реальная сложность, которую вызывающий код всё равно вынужден знать — только теперь уже без помощи компилятора.

### 3.1. *IBonusRepository.ReHold* — четыре числа без имён

```csharp
Task<(int currentHold, int newHoldValue, int delta, string status)> ReHold(Calculation calculation);
```

Четыре значения в кортеже — это уже объект. Просто безымянный и без методов. Но проблема не в количестве — проблема в *string status*. Что туда приходит? Судя по использованию, это что-то вроде *"increased"*, *"decreased"*, *"unchanged"*. Но это нигде не написано. Вызывающий код либо сравнивает строки вслепую, либо лезет в реализацию *ReHold*, чтобы понять, какие значения вообще возможны.

Компилятор тут бессилен. Напиши *"Increased"* вместо *"increased"* — и сравнение молча провалится.

```csharp
public record ReHoldResult(int CurrentHold, int NewHoldValue, int Delta, ReHoldStatus Status);

public enum ReHoldStatus { Increased, Decreased, Unchanged, Blocked }

Task<ReHoldResult> ReHold(Calculation calculation);
```

Теперь интерфейс «сложнее» — надо знать *ReHoldStatus*. Но эта сложность уже существовала: просто раньше она была невидима, а теперь видна. И IDE подсказывает варианты, и компилятор ловит опечатки.

---

### 3.2. *Calculation.Messages* — список с паролем внутри

```csharp
public List<(string type, string message)> Messages { get; private set; }
```

Выглядит просто. Но откроем *MessageFactory* — и увидим, что *type* принимает строго определённые значения: *"max"*, *"vk"*, *"birthday"*. Это не произвольные строки — это каналы отправки уведомлений. По сути, enum, зашитый в *string*.

Кто-то добавит новый канал и напишет *"telegram"* в одном месте и *"Telegram"* в другом — и сообщение потеряется. Никакой ошибки, никакого предупреждения. Просто гость не получит уведомление в день рождения.

```csharp
public enum MessageChannel { Max, Vk, Birthday, Antifraud }

public record CalculationMessage(MessageChannel Channel, string Text);

public List<CalculationMessage> Messages { get; private set; }
```

Список стал чуть «тяжелее» — но теперь добавить новый канал можно только через enum, а значит через явное решение, которое компилятор потребует обработать во всех switch-выражениях.

---

### 3.3. *ConditionFactory.Create* — *string name* как секретный пароль

```csharp
public Condition Create(string name, string data)
{
    string typeFullName = $"{_namespacestring}.{name}";
    Condition result = (Condition)JsonSerializer.Deserialize(data, Type.GetType(typeFullName), ...);
    result.InitDb(_conditionContext);
    return result;
}
```

Параметр *string name* — это не просто имя. Это полное имя C#-класса в пространстве *Loyalty.Domain.Conditions*, которое будет передано в *Type.GetType()*. Передай *"GuestSex"* — работает. Передай *"guestsex"* — *NullReferenceException*. Передай *"NonExistentCondition"* — тоже *NullReferenceException*, только в другом месте.

Всё это знание живёт только внутри метода. Снаружи видна просто строка.

Здесь нельзя убрать сложность полностью — рефлексия по имени типа это намеренное решение для динамической загрузки условий из JSON. Но можно хотя бы сделать контракт честным:

```csharp
/// <summary>
/// Создаёт условие по имени C#-класса из пространства Loyalty.Domain.Conditions.
/// name должен точно совпадать с именем класса, включая регистр: "GuestSex", "OrderAmount".
/// Если класс не найден — выбрасывает InvalidOperationException (не NullReferenceException).
/// data — JSON с полями условия, десериализуется в найденный тип.
/// </summary>
public Condition Create(string name, string data)
{
    string typeFullName = $"{_namespacestring}.{name}";
    var type = Type.GetType(typeFullName)
        ?? throw new InvalidOperationException(
            $"Тип условия '{name}' не найден. Ожидается имя класса из {_namespacestring}.");

    var result = (Condition)JsonSerializer.Deserialize(data, type, ...);
    result.InitDb(_conditionContext);
    return result;
}
```

Реализация осталась той же. Но теперь: ошибка вменяемая, контракт параметра описан, и следующий разработчик не потратит час на отладку *NullReferenceException* без стектрейса.

---

## Что я вынес из этого

Все девять примеров объединяет одно: **сложность никуда не делась, её просто спрятали**.

*calculation = null* перед двумя *if* — это не упрощение логики, это перенос логики о приоритете в голову читателя. *string status* в кортеже *ReHold* — это не простой интерфейс, это enum без подсказок IDE. *birthday >= startDate* — это не баг, это документ, который не написали, и он обошёлся тем, что акции ко дню рождения не работали.

Самое неприятное во всех этих случаях — они не выглядят как проблема. Код компилируется. Тесты проходят. Ревью принимает. Проблема проявляется позже: в продакшене, в отладке в два часа ночи, в вопросе «а почему гости не получают поздравления?»

Хороший принцип, который я для себя сформулировал: если чтобы понять контракт функции, нужно читать её реализацию — контракт не написан. Имя, тип параметра, XML-doc, enum вместо строки — всё это инструменты, чтобы контракт жил снаружи, а не внутри. Тогда и читать проще, и менять безопаснее, и отлаживать быстрее.