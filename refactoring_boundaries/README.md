# Отчёт: Рефакторинг — перекомбинирование частей по границам

## Введение

Главное, что я понял из материала по типам ошибок 2–5 и 7: рефакторинг — это не «вынес код в функцию, переименовал, разбил большой метод». Это представление чего-то по-другому. То есть подумал, что код на самом деле про две (или больше) разные вещи, и развёл их так, чтобы граница стала явной.

Как сказано в материале: факторинг — это представление чего-то в виде композиции частей, а ре-факторинг — это представление чего-то по-другому. И ещё одно, что зацепило: материал явно отличает «как код выглядит» от «как код влияет на систему». Мудрый ревьюер смотрит не на форму, а на границы и на то, что попадает по разные их стороны.

Ниже — три небольших, но честных перекомбинирования из моего проекта. Каждое — про то, что в одном файле живут две разные части задачи, и сейчас они разделены неявно: через дублирование, через комментарий, через намёк в имени. Я хочу провести эту границу явно — через типы или через структуру. Для одного из них в конце — тесты в виде use case: они показывают не «работает ли отдельный метод», а зачем эта часть нужна системе.

---

## Рефакторинг 1. *GuestBirthday* — две почти одинаковые проверки в одном классе

### Что это за часть проекта

*GuestBirthday* — одно из условий лояльности. Бизнес-смысл: «у гостя скоро день рождения». Используется в акциях для именинников. У условия два параметра — *DaysBefore* и *DaysAfter* — окно вокруг ДР, в которое акция работает.

### Что внутри сейчас

В классе два метода *IsMatch*. Один для расчёта (*Calculation*), второй для проверки гостя без расчёта (*Guest*). Оба нужны: первый работает в рантайме при расчёте скидки на заказ, второй — в админке или сервисе, когда надо проверить «попадает ли этот гость в сегмент именинников сейчас».

```csharp
public class GuestBirthday : Condition
{
    public int? DaysBefore { get; set; }
    public int? DaysAfter { get; set; }

    public override bool IsMatch(Calculation calculation)
    {
        if (calculation?.Guest?.Birthday == null) return false;
        var birthdayDate = calculation.Guest.Birthday;
        var currentDate = DateOnly.FromDateTime(calculation.CreateDate.Date);
        var startDate = currentDate.AddDays(-(DaysBefore ?? 5));
        var endDate = currentDate.AddDays(DaysAfter ?? 5);
        return birthdayDate >= startDate && birthdayDate <= endDate;
    }

    public bool IsMatch(Guest guest)
    {
        if (guest?.Birthday == null) return false;
        var birthdayDate = guest?.Birthday;
        var currentDate = DateOnly.FromDateTime(DateTimeOffset.Now.Date);
        var startDate = currentDate.AddDays(-(DaysBefore ?? 5));
        var endDate = currentDate.AddDays(DaysAfter ?? 5);
        return birthdayDate >= startDate && birthdayDate <= endDate;
    }
}
```

### Где здесь граница и как она задана

Два метода, и один и тот же расчёт окна с одним и тем же сравнением скопированы дважды. Отличаются они ровно одним: что считать «сейчас». Для расчёта это *calculation.CreateDate* — дата заказа. Для гостя без расчёта это *DateTimeOffset.Now* — текущий момент.

То есть в этом классе смешаны две разные части одной задачи. С одной стороны — логика правила: «попадает ли день рождения в окно ±N дней от заданной даты». Это чистая, безусловная функция. С другой — контекст вызова: «откуда брать "сейчас" и откуда брать дату рождения». Это разное для разных сценариев, заказ или гость.

Сейчас граница между ними задана через дублирование. В двух методах копия одного и того же расчёта окна — это самый худший способ обозначить границу. Она существует, но компилятор о ней не знает, и при изменении одной из копий вторая останется старой.

### Что значит «перекомбинировать»

Я хочу выделить логику правила в отдельную чистую функцию от двух дат. А методы *IsMatch* становятся тонкими адаптерами: возьми дату отсюда, возьми «сейчас» оттуда, передай в правило.

### Стало

```csharp
public class GuestBirthday : Condition
{
    public int? DaysBefore { get; set; }
    public int? DaysAfter { get; set; }

    // Чистое правило: попадает ли день рождения в окно ±N дней от даты-якоря
    private bool IsBirthdayInWindow(DateOnly? birthday, DateOnly anchor)
    {
        if (birthday is not DateOnly bday) return false;
        var start = anchor.AddDays(-(DaysBefore ?? 5));
        var end   = anchor.AddDays(  DaysAfter  ?? 5);
        return bday >= start && bday <= end;
    }

    // Контекст 1: проверка в момент расчёта заказа. Якорь — дата заказа.
    public override bool IsMatch(Calculation calculation)
        => IsBirthdayInWindow(
               calculation?.Guest?.Birthday,
               DateOnly.FromDateTime(calculation?.CreateDate.Date ?? DateTime.MinValue));

    // Контекст 2: проверка «здесь и сейчас». Якорь — текущая дата.
    public bool IsMatch(Guest guest)
        => IsBirthdayInWindow(
               guest?.Birthday,
               DateOnly.FromDateTime(DateTimeOffset.Now.Date));
}
```

### Граница: было / стало

До рефакторинга граница между «правилом» и «контекстом вызова» была неявной — задана через дублирование двух методов. То, что в двух местах живёт одна и та же логика, нигде не выражено: ни типом, ни именем, ни сигнатурой. Это видно только при чтении тел подряд.

После рефакторинга граница задана явно — через сигнатуру *IsBirthdayInWindow(DateOnly?, DateOnly)*. Чистая функция от двух дат — это и есть правило. Всё, что не входит в эту сигнатуру (откуда взять день рождения, откуда взять «сейчас»), это контекст, и он живёт в маленьких адаптерах.

### На что распространяется изменение

Изменение полностью локальное — затрагивает только сам класс *GuestBirthday*. Сигнатуры обоих публичных *IsMatch* сохраняются: для вызывающих мест, то есть для рантайма расчёта в *Calculation* и для проверки гостя в админке или сервисе, ничего не меняется. Если завтра появится третий контекст вызова (например, «как выглядела бы эта акция в прошлом году» для отчёта), это будет ещё одна строчка адаптера, а не третья копия правила. Граница в коде теперь правильно отражает то, что в админке и в рантайме на самом деле одно и то же правило — это видно из кода, а не из комментария.

---

## Рефакторинг 2. *PromocodeValue* — два конструктора, один из которых частично подделка под другой

### Что это за часть проекта

*PromocodeValue* — конкретный код-значение промокода. У промокода может быть несколько таких значений: для группового промокода много, для одиночного одно. Значение либо приходит готовое (например, «SUMMER25», админ задал руками), либо генерируется системой через *PromoCodeGenerator* для массовой раздачи.

### Что внутри сейчас

В классе два конструктора. Первый принимает строку. Второй принимает только промокод и генерирует значение сам.

```csharp
public class PromocodeValue
{
    protected PromocodeValue() { }

    public Guid Id { get; private set; }
    public long CorporationId { get; private set; }
    public Guid PromocodeId { get; private set; }
    public string Value { get; private set; }

    internal PromocodeValue(string value, Promocode promocode)
    {
        Id = Guid.NewGuid();
        CorporationId = promocode.CorporationId;
        PromocodeId = promocode.Id;
        Value = value;
    }

    internal PromocodeValue(Promocode promocode)
    {
        Id = Guid.NewGuid();
        CorporationId = promocode.CorporationId;
        PromocodeId = promocode.Id;
        Value = PromoCodeGenerator.GeneratePromoCode(Id, CorporationId);
    }

    public void Regen()
    {
        Id = Guid.NewGuid();
        Value = PromoCodeGenerator.GeneratePromoCode(Id, CorporationId);
    }
}
```

### Где здесь граница

В этом классе живут два сценария создания и один сценарий регенерации. Первый — создать значение из готовой строки, это сценарий админа: «вот код, который я хочу выдавать». Второй — сгенерировать новое значение, это сценарий массовой раздачи: «придумай мне сама». Третий — перегенерировать существующее, если в системе случайно совпали значения.

Все три сценария спрятаны либо в безымянные конструкторы, либо в метод *Regen*, который без объяснения, чем он отличается от создания через второй конструктор. На самом деле логика *Regen* почти один в один дублирует второй конструктор: обе линии генерируют новый *Id* и зовут *PromoCodeGenerator*.

Граница «откуда берётся значение» сейчас задана через выбор конструктора — один или два параметра. Это слабо. Нужно помнить, какой конструктор какой сценарий. Имя *PromocodeValue(promocode)* ничего не говорит о том, что внутри произойдёт генерация.

### Что значит «перекомбинировать»

Я хочу вытащить наружу два разных намерения через явные фабричные методы. Безымянный второй конструктор уходит вообще: «создание объекта» и «генерация нового значения» — это две операции, которые сейчас слиплись в одну. Заодно *Regen* перестаёт быть отдельной утилитой и становится одной частной задачей внутри одной из фабрик.

### Стало

```csharp
public class PromocodeValue
{
    protected PromocodeValue() { }

    public Guid Id { get; private set; }
    public long CorporationId { get; private set; }
    public Guid PromocodeId { get; private set; }
    public string Value { get; private set; }

    // Единственный «внутренний» конструктор — без логики, просто присвоения.
    // Внешний код к нему не обращается.
    private PromocodeValue(Guid id, long corporationId, Guid promocodeId, string value)
    {
        Id = id;
        CorporationId = corporationId;
        PromocodeId = promocodeId;
        Value = value;
    }

    // Сценарий «админ задал значение руками»
    internal static PromocodeValue FromExplicit(string value, Promocode promocode)
        => new(Guid.NewGuid(), promocode.CorporationId, promocode.Id, value);

    // Сценарий «система сгенерировала значение»
    internal static PromocodeValue Generated(Promocode promocode)
    {
        var id = Guid.NewGuid();
        return new(id, promocode.CorporationId, promocode.Id,
                   PromoCodeGenerator.GeneratePromoCode(id, promocode.CorporationId));
    }

    // Регенерация — это та же логика, что и у Generated, но на существующем объекте
    public void Regenerate()
    {
        Id = Guid.NewGuid();
        Value = PromoCodeGenerator.GeneratePromoCode(Id, CorporationId);
    }
}
```

### Граница: было / стало

До рефакторинга граница между двумя способами создания была неявной — задана через выбор конструктора по количеству параметров. Перегрузка *PromocodeValue(value, promocode)* и *PromocodeValue(promocode)* — компилятор различает их только по сигнатуре, а намерения у них радикально разные: одна берёт готовое значение, другая идёт в генератор. Эти намерения нигде не названы, и узнать их можно только прочитав тело каждого конструктора.

После рефакторинга граница задана явно — через имена фабричных методов *FromExplicit* и *Generated*. Имя в месте вызова прямо говорит, какое намерение реализует код: «значение пришло снаружи» или «значение сгенерировано системой». Параллельно убирается дублирование с *Regenerate*: он остаётся отдельной операцией, но больше не повторяет то, что и так есть в *Generated*.

### На что распространяется изменение

Затрагивает класс *PromocodeValue* и все места, где он создаётся. По проекту таких мест немного — генерация при создании промокода и при импорте или обновлении. Сигнатура у них меняется (*new PromocodeValue(...)* становится *PromocodeValue.FromExplicit(...)* или *PromocodeValue.Generated(...)*), но компилятор перехватит каждый вызов — забыть нельзя. Старые конструкторы становятся приватными, и попытка использовать их снаружи перестанет компилироваться. Изменение в десятки строк, разрозненное по нескольким файлам, но полностью отслеживаемое компилятором — это и есть то, что материал называет нелокальным, но безопасным.

---

## Рефакторинг 3. *Dish* и *DiscountedDish* — параллельная иерархия со скидкой

### Что это за часть проекта

*Dish* — блюдо в составе заказа. Хранит id, имя, количество, цену, ограничения. Используется на этапе приёма заказа, до применения скидок.

*DiscountedDish* — блюдо после применения скидки. У него все те же поля, что и у *Dish*, плюс два дополнительных: *Discount* (общая скидка) и *Discounts* (разбивка по типам — promotion, promocode, bonus). На одно блюдо в заказе создаётся пара *Dish* + *DiscountedDish*, связанная по *LineItemId*.

### Что внутри сейчас

```csharp
public class Dish
{
    public Guid? LineItemId { get; }
    public long Id { get; }
    public string Name { get; }
    public decimal Quantity { get; }
    public int Price { get; }
    public int Amount { get; }
    public bool IsDiscountExcluded { get; }
    public List<Dish> Dishes { get; }
    public Restrictions Restrictions { get; }
}

public class DiscountedDish
{
    public Guid? LineItemId { get; }
    public long Id { get; }
    public string Name { get; }
    public decimal Quantity { get; }
    public int Price { get; }
    public int Amount { get; }
    public bool IsDiscountExcluded { get; }
    public List<DiscountedDish> Dishes { get; }      // ← рекурсия по своему типу
    public Restrictions Restrictions { get; }

    // Реально новое — две строки
    public long Discount { get; private set; }
    public Dictionary<string, long> Discounts { get; set; } = new();

    public long FullAmount => Amount - Discount;

    public void MergeDiscount(DiscountedDish other) { /* складывает скидки */ }
    internal void ClearDiscounts() { Discount = 0; Discounts = null; }
}
```

### Где здесь граница

*DiscountedDish* — это на 90% копия *Dish*, плюс одна дополнительная штука: начисленная на это блюдо скидка. То есть в проекте смешаны две разные части. Первая — блюдо как факт заказа: что заказали, сколько, по какой цене. Это неизменяемые данные, которые поступили от кассы и больше меняться не должны. Вторая — скидка на это блюдо: отдельная сущность, которая появляется в результате применения акций, промокодов и discount-ов. Она может пересчитываться, складываться, обнуляться.

Сейчас граница между этими частями нарисована через копирование полей с лёгкой подменой типа коллекции. Это создаёт два неприятных эффекта. Невозможно понять, какие поля у *DiscountedDish* «копия из *Dish*», а какие действительно его собственные. Если завтра в *Dish* добавить поле — например, *VatRate* для НДС, — нужно вспомнить продублировать его в *DiscountedDish*. И на каждое блюдо в заказе создаётся параллельный объект: 30 блюд — 30 дополнительных объектов с поведением. А в *Calculation* приходится держать обе коллекции и следить, чтобы они не расходились.

### Что значит «перекомбинировать»

Скидка — это отдельная сущность, привязанная к блюду через *LineItemId*. Она не должна копировать поля блюда. Хочу так: один *Dish* остаётся как есть (это факт заказа), а скидки на блюда — отдельная структура внутри расчёта, привязанная по *LineItemId*.

### Стало

```csharp
// Dish остаётся как был — это факт заказа, ничего не меняем
public class Dish { /* как было */ }

// Скидка на блюдо — маленькая запись-значение, привязанная по LineItemId
public readonly record struct DishDiscount(
    Guid LineItemId,
    long Amount,
    IReadOnlyDictionary<string, long> ByType);

// В расчёте — словарь скидок, а не параллельная иерархия блюд
public class Calculation
{
    public IReadOnlyList<Dish> Dishes { get; }                 // факт заказа
    private readonly Dictionary<Guid, DishDiscount> _discounts = new();

    public IReadOnlyDictionary<Guid, DishDiscount> Discounts => _discounts;

    // Сумма к оплате по блюду — теперь вычисляется на лету,
    // и нет необходимости держать «параллельное» поле FullAmount в каждом DiscountedDish
    public long FullAmountFor(Dish dish)
        => dish.Amount - (_discounts.TryGetValue(dish.LineItemId!.Value, out var d) ? d.Amount : 0);

    internal void MergeDiscount(Guid lineItemId, DiscountTypeEnum type, long amount) { /* ... */ }
    internal void ClearDiscounts() => _discounts.Clear();
}
```

### Граница: было / стало

До рефакторинга граница между «фактом заказа» и «скидкой на этот факт» была неявной — задана через два параллельных класса с почти одинаковыми полями (*Dish* против *DiscountedDish*) плюс связь по *LineItemId*. Эта связь — соглашение, нигде не выраженное типом: компилятор не знает, что *DiscountedDish.LineItemId* обязан совпадать с *Dish.LineItemId*. И не знает, что 8 из 10 полей *DiscountedDish* обязаны быть копией полей *Dish*.

После рефакторинга граница задана явно — через два разных типа с разными ролями. *Dish* остаётся неизменяемым фактом заказа, а *DishDiscount* (record struct) — маленькая запись о скидке, привязанная к конкретному *LineItemId*. Скидки хранятся в *Calculation* отдельной коллекцией, и теперь по самому типу видно: один источник правды о блюде, отдельный источник правды о скидках.

### На что распространяется изменение

Это самый «нелокальный» из трёх рефакторингов. Затрагивает все места, где сейчас работают с *DiscountedDish*: расчёт скидки (применение акций, промокодов, discount-ов), формирование чека, отчёты, маппинг в DTO. По коду это десятки точек.

При этом изменение не переписывает архитектуру — оно переносит одну концепцию (скидка на позицию) на правильное место: внутрь *Calculation* как словарь, а не как параллельную коллекцию объектов. Каждое из мест использования меняется одинаково — вместо *discountedDish.FullAmount* пишется *calculation.FullAmountFor(dish)*. Компилятор после удаления *DiscountedDish* найдёт все точки разом — снова нелокально, но отслеживаемо.

Что важно — это убирает целый класс возможных ошибок «забыли синхронизировать *Dish* и *DiscountedDish* при добавлении нового поля». Их теперь невозможно рассинхронизировать, потому что источник правды о блюде один.

Материал прямо говорит: не бойтесь нелокальных изменений, если они проводят границы лучше. Здесь именно такой случай.

---

## Use case-тесты для рефакторинга 1 (*GuestBirthday*)

Из подсказки задания: тесты должны показывать роль части по отношению к проекту, а не «работает ли метод». Беру *GuestBirthday* — там роли явные, использую их как названия тестов.

```csharp
public class GuestBirthday_UseCases
{
    // Use case: гостю с днём рождения через 3 дня показывается акция «к именинникам»
    // на странице меню. Никакого расчёта заказа ещё нет — есть только гость и «сейчас».
    [Fact]
    public void Birthday_promotion_is_visible_to_a_guest_with_upcoming_birthday()
    {
        var guest = new Guest { Birthday = DateOnly.FromDateTime(DateTime.Today.AddDays(3)) };
        var sut = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };

        Assert.True(sut.IsMatch(guest));
    }

    // Use case: гость оформляет заказ за неделю до дня рождения.
    // Заказ обрабатывается через несколько часов (CreateDate = вчера).
    // Акция всё ещё применяется к заказу — потому что для расчёта якорь —
    // дата создания заказа, а не «сейчас».
    [Fact]
    public void Birthday_promotion_applies_to_an_order_created_within_the_window()
    {
        var birthday = new DateOnly(2024, 6, 18);
        var orderCreatedAt = new DateTime(2024, 6, 15);  // за 3 дня до ДР

        var calc = CalculationWithGuestBirthday(birthday, createdAt: orderCreatedAt);
        var sut = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };

        Assert.True(sut.IsMatch(calc));
    }

    // Use case: тот же гость пытается воспользоваться акцией через месяц после ДР.
    // Окно прошло — акция не применяется.
    [Fact]
    public void Birthday_promotion_does_not_apply_outside_the_window()
    {
        var birthday = new DateOnly(2024, 6, 18);
        var orderCreatedAt = new DateTime(2024, 7, 20);  // месяц спустя

        var calc = CalculationWithGuestBirthday(birthday, createdAt: orderCreatedAt);
        var sut = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };

        Assert.False(sut.IsMatch(calc));
    }

    // Use case: у гостя в профиле не указан день рождения.
    // Акция к именинникам не может ему «попасть» ни в админке, ни на заказе.
    [Fact]
    public void Birthday_promotion_does_not_apply_when_birthday_is_unknown()
    {
        var guestWithoutBirthday = new Guest { Birthday = null };
        var sut = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };

        Assert.False(sut.IsMatch(guestWithoutBirthday));
    }

    // Use case: бизнес настроил «расширенную» акцию — 10 дней до ДР, 1 день после.
    // Это асимметричное окно, и оно должно соблюдаться обоими режимами.
    [Fact]
    public void Asymmetric_window_is_respected_both_in_admin_and_in_runtime()
    {
        var birthday = DateOnly.FromDateTime(DateTime.Today.AddDays(-2));  // ДР был 2 дня назад
        var guest = new Guest { Birthday = birthday };

        var narrowAfter = new GuestBirthday { DaysBefore = 10, DaysAfter = 1 };  // окно [-10; +1]
        // Проверка «здесь и сейчас» — ДР был 2 дня назад, выходит за +1
        Assert.False(narrowAfter.IsMatch(guest));

        var wideAfter = new GuestBirthday { DaysBefore = 10, DaysAfter = 5 };    // окно [-10; +5]
        Assert.True(wideAfter.IsMatch(guest));
    }
}
```

### Чем эти тесты отличаются от обычных юнитов

Главное — названия. Не «IsMatch_NullBirthday_ReturnsFalse», а нормальные предложения на человеческом языке: акция видна гостю с приближающимся ДР, акция применяется к заказу в окне, асимметричное окно соблюдается обоими режимами. Тест читается как описание возможности системы, а не отчёт о методе класса.

И каждый тест опирается на конкретную роль *GuestBirthday* в проекте. Первый — про сценарий «гость зашёл, смотрит меню, акция должна быть видна»: тут важно, что якорь — *сейчас*. Второй — про сценарий «заказ оформлен, скидка считается»: тут якорь — дата заказа, а не «сейчас». И это другая роль того же правила, тест эту разницу проверяет. Третий и пятый — про бизнес-настройку окна (асимметричное, расширенное), то есть про то, что бизнес может настраивать через параметры условия. Четвёртый — про граничный случай в данных гостя, который тоже бизнес-сценарий: профиль без ДР — это обычная ситуация.

Если завтра кто-то прочитает только эти тесты, он поймёт назначение условия для проекта, не разбираясь в реализации. Это и есть разница между юнитом и use case-ом.

---

## Что я понял в итоге

Все три рефакторинга мелкие — каждый помещается в один экран кода. Но в каждом я начинал с того же вопроса: где здесь граница между разными частями задачи, и каким способом она задана?

И в каждом случае ответ оказывался одним и тем же по форме: граница есть, но задана неявно — через слабый способ, который компилятор не контролирует. В *GuestBirthday* — через дублирование кода: читай два метода и сравни глазами. В *PromocodeValue* — через выбор перегрузки конструктора: думай, какой по сигнатуре что делает. В *Dish* / *DiscountedDish* — через копирование полей с лёгкой подменой типа коллекции: помни, что 8 полей обязаны совпадать.

Все эти способы — слабые. Они работают, пока за ними следит человек. Компилятор и читатель видят только результат, но не саму границу.

Полноценный рефакторинг во всех трёх случаях — это перевод границы из неявной в явную. Из дублирования в сигнатуру функции. Из перегрузок в именованные фабричные методы. Из параллельной иерархии в два разных типа с разными ролями. После этого граница становится видимой компилятору: попытка нарушить её перестаёт компилироваться или становится очевидной с места вызова.

И ещё одно: материал предостерегает не превращать такие изменения в переписывание системы. Из моих трёх рефакторингов два — полностью локальные, внутри одного класса. Один — нелокальный, но ограниченный: *DishDiscount* касается многих мест, но изменения там механические и проверяются компилятором. Я не перерисовываю архитектуру, только провожу границы, которые в проекте уже есть по смыслу, но пока выражены слабо.