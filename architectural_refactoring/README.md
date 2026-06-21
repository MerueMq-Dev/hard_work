# Отчёт: Два варианта крупных нелокальных изменений в проекте

## Введение

Шестой тип ошибок из материала — про страх перед нелокальными изменениями. Видишь проблему — и вместо того, чтобы перекомпоновать систему, выделяешь ещё один helper, добавляешь ещё один класс рядом со старым, переносишь проверки в отдельный метод. По сути та же грязь, просто переложенная в другую коробку. Настоящий рефакторинг — это правка, которая идёт через много мест разом, но не превращается в переписывание системы с нуля.

В моём проекте (бэкенд системы лояльности для ресторанной сети) есть два направления, где такие изменения назрели. Они разные по природе: первое — **разделить** то, что сейчас слиплось, второе — наоборот, **схлопнуть** три почти одинаковые копии в одну. Оба про дизайн, а не про правку метода.

---

## Изменение 1. Разделить проект на два bounded context-а

### Что сейчас

Проект по Лояльности, и под этим именем живёт всё подряд: правила лояльности (скидки, акции, промокоды, сегменты гостей), импорт меню из внешних кассовых систем, корпорации и рестораны как сущности, расчёт скидки на заказ, бонусные кошельки, саги координации, интеграции с мессенджерами. Всё в одной структуре проектов *Loyalty.Domain*, *Loyalty.Infrastructure*, *Loyalty.WebApi*, переплетено зависимостями вдоль и поперёк.

### Что не так

Внутри на самом деле живут две разные области, случайно оказавшиеся в одном репозитории. Первая — собственно лояльность: правила, по которым на заказ начисляется скидка. Вторая — каталог: что вообще продаётся, какое меню в каком ресторане, какие цены. Они связаны по данным, но не по логике. Лояльности незачем знать, что меню грузится сагой из внешней кассы. Импорту меню незачем знать про бонусы гостя.

Из-за этого смешения новичок, пришедший «поправить расчёт скидок», вынужден разбираться в сагах импорта. И правка в *Dish* теоретически может уронить расчёт скидки — между ними нет видимой границы, только переплетение.

### Что я хочу сделать

Провести между ними явную границу и зафиксировать её на уровне раздельных проектов. **Loyalty Core** — всё про правила: *Calculation*, *Condition*, *Reward*, *Discount*/*Promotion*/*Promocode*, *Guest*, бонусы. **Catalog & Operations** — всё про реальность: *Restaurant*, *Corporation*, *Menu*, *Dish*, *Product*, импорт. Между ними узкий контракт: лояльность принимает готовые данные о позиции и понятия не имеет, откуда они взялись.

### Почему это нелокально

Затронуто будет всё: репозитории, саги, DTO, регистрация сервисов. Это десятки файлов и сотни мест. Поэтому важно не скатиться в «давайте перепишем заново»: сначала ввести интерфейсы пограничного API, потом аккуратно переносить типы по одному.

### Чем это лучше

Каждый контекст можно понимать отдельно — не надо больше держать в голове весь проект, чтобы поправить одну вещь. Граница становится одна, тщательно спроектированная — а не двести точек переплетения. И каталог при случае можно вынести в отдельный сервис, потому что он уже не привязан к лояльности изнутри.

### Как это выглядит в коде

Сейчас *Calculation* тащит за собой типы каталога напрямую:

```csharp
// Сейчас, Loyalty.Domain — лояльность зависит от Dish из каталога
public class Calculation
{
    public List<Dish> Dishes { get; private set; }              // тип из каталога
    public List<DiscountedDish> DiscountedDishes { get; }       // параллельная иерархия
    public Guest Guest { get; private set; }

    internal void SetGuest(Guest guest, ...)
    {
        // расчёт знает про телефон, идентификаторы в мессенджерах, антифрод
        GuestPhone = guest?.Phone;
        ChatId = guest?.ChatId;
        // ...
    }
}
```

После разделения у каждого контекста — своя сборка. Контакт между ними через узкий порт:

```csharp
// Loyalty.Core — пограничный контракт
public interface ICatalogPort
{
    Task<OrderLine?> GetLine(long restaurantId, Guid lineItemId);
}

public record OrderLine(Guid LineItemId, long ItemId, string Name, int Price, int Amount);

// Расчёт принимает уже подготовленные строки
public class Calculation
{
    public IReadOnlyList<OrderLine> Lines { get; }
    public IReadOnlyDictionary<Guid, DishDiscount> Discounts { get; }
}

// Catalog.Adapters.Loyalty — реализация порта на другой стороне границы
public class CatalogLoyaltyAdapter : ICatalogPort
{
    public async Task<OrderLine?> GetLine(long restaurantId, Guid lineItemId)
    {
        var product = await _menuRepository.GetByLine(restaurantId, lineItemId);
        return product is null ? null
            : new OrderLine(lineItemId, product.Id, product.Name, product.PriceMinor, 1);
    }
}
```

Самое важное — после разделения *Calculation* физически не может дотянуться до *Dish* или адаптера кассы. Этих типов нет в её сборке. Граница охраняется не дисциплиной, а компилятором.

---

## Изменение 2. Свести *Discounts*/*Promotions*/*Promocodes* в одну сущность *Benefit*

### Что сейчас

В системе три типа инструментов лояльности. **Скидки** — постоянные правила «при условии Х — скидка Y». **Акции** — то же самое плюс временное окно и лимит. **Промокоды** — то же самое плюс активация конкретным кодом. Каждый — отдельный модуль со своей сущностью, репозиторием, командами Create/Edit/Delete, хэндлерами.

### Что не так

Если открыть три класса рядом — они на 80% одно и то же, описанное три раза. Везде *Name*, *Description*, *IsActive*, *IsCombined*, *Conditions*, *Reward*. Везде одни и те же операции. Отличия узкие и точечные: у акции — даты и лимит, у промокода — список кодов, у скидки этих штук нет.

Каждое новое требование — добавить поле, новый тип награды, флаг — это три правки в трёх местах. Промахнуться в одной из них слишком легко.

### Что я хочу сделать

Выделить одну сущность *Benefit* и сделать три текущих типа её вариантами. Общие поля и операции переезжают в *Benefit*. Специфика — в маленькую структуру *BenefitActivation* с тремя подтипами: «активно всегда», «временное окно с лимитом», «активация по коду». Команды Create-Edit сводятся в одну с активацией как параметром.

И отдельно — один репозиторий вместо трёх. Сейчас при расчёте скидки делается три запроса к трём таблицам, потом результаты сшиваются. После объединения — один запрос с фильтром по типу активации. Быстрее и сильно проще.

### Почему это нелокально

Затронуто три модуля целиком: репозитории, хэндлеры, контроллеры, DTO, тесты. Плюс схема БД — одна общая таблица плюс две специальные. Плюс миграция данных. Плюс публичные API контроллеров. Объём большой, даже если делать аккуратно.

### Чем это лучше

Стоимость нового требования падает втрое. Бизнес говорит «давайте тегировать инструменты» — это одна правка в *Benefit*, а не три симметричные с риском забыть одну. И ещё одно: становится понятно, что это не три разные сущности, а **три способа активировать одну модель**. Бизнес, кстати, так и думает — для них это всё «правила, по которым клиент получает выгоду», и наше деление на скидки/акции/промокоды для них неочевидно.

### Как это выглядит в коде

Сейчас три класса с почти одинаковыми полями:

```csharp
// Loyalty.Domain.Discounts.Discount
public class Discount {
    public string Name { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsCombined { get; private set; }
    public List<Condition> Conditions { get; set; }
    public Reward Reward { get; set; }
}

// Loyalty.Domain.Promotions.Promotion — то же + окно и лимит
public class Promotion {
    public string Name { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsCombined { get; private set; }
    public List<Condition> Conditions { get; set; }
    public Reward Reward { get; set; }
    public DateTimeOffset? Start { get; private set; }   // ← специфика
    public DateTimeOffset? End { get; private set; }     // ← специфика
    public int? Limit { get; private set; }              // ← специфика
}

// Loyalty.Domain.Promocodes.Promocode — то же + список кодов
public class Promocode {
    public string Name { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsCombined { get; private set; }
    public List<Condition> Conditions { get; set; }
    public Reward Reward { get; set; }
    public List<PromocodeValue> Values { get; }          // ← специфика
}
```

После объединения — общее ядро плюс маленькая структура активации:

```csharp
public class Benefit
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsCombined { get; private set; }
    public List<Condition> Conditions { get; private set; }
    public Reward Reward { get; private set; }
    public BenefitActivation Activation { get; private set; }   // ← здесь вся разница

    public bool IsApplicableAt(DateTimeOffset moment)
        => IsActive && Activation.IsActiveAt(moment);
}

public abstract record BenefitActivation
{
    public abstract bool IsActiveAt(DateTimeOffset moment);

    public sealed record Always : BenefitActivation
    {
        public override bool IsActiveAt(DateTimeOffset moment) => true;
    }

    public sealed record TimeWindow(DateTimeOffset? Start, DateTimeOffset? End, int? Limit, int Used)
        : BenefitActivation
    {
        public override bool IsActiveAt(DateTimeOffset moment)
            => (Start is null || moment >= Start)
            && (End   is null || moment <= End)
            && (Limit is null || Used < Limit);
    }

    public sealed record PromoCodes(IReadOnlyList<PromocodeValue> Values) : BenefitActivation
    {
        public override bool IsActiveAt(DateTimeOffset moment) => true;
    }
}
```

Три команды сводятся в одну:

```csharp
// Было — три параллельные команды
public class CreateEmptyPromotionRequest : IRequest<PromotionDto> { /* ... */ }
public class CreatePromocodeRequest      : IRequest<PromocodeDto> { /* ... */ }
public class CreateDiscountRequest       : IRequest<DiscountDto>  { /* ... */ }

// Стало — одна команда, активация как параметр
public class CreateBenefitRequest(
    string name, bool isActive, bool isCombined,
    BenefitActivation activation,
    List<(string name, dynamic options)> conditions,
    (string name, dynamic options) reward
) : IRequest<BenefitDto>;
```

---

## Вывод

Оба изменения меняют дизайн системы, а не код в файлах. И оба родились не из «увидел корявый метод», а из **повторяющейся боли**: каждое новое требование стоит втрое дороже, чем должно. Это и есть признак того, что нужна перекомпоновка, а не косметика.

Оба пугают объёмом — и это ровно та самая ошибка из материала. Страх заставляет делать вместо нелокальной правки что-то маленькое и безопасное, а проблема накапливается. Здесь важно держать золотую середину: не «давайте перепишем заново», но и не «давайте оставим как есть». Двигаться по понятным шагам, без обрушения текущей функциональности, и на каждом этапе иметь чёткую цель.

Если выбирать, что делать первым — я бы начал со второго. Оно ограниченнее по охвату, граница понятнее, и выигрыш в скорости новых требований заметен на каждой следующей правке. Первое — фундаментальнее, и его лучше делать после, когда команда уже привыкла к мысли, что в проекте можно проводить структурные правки, а не только дописывать новые файлы рядом со старыми.