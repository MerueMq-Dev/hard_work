# Рефакторинг с паттерном Visitor

## Введение

Сначала я искал пример «неистинного» наследования, чтобы попробовать применить паттерн Visitor. Взял первый попавшийся код с переопределением методов, но быстро понял, что это не то, что нужно.

Visitor действительно имеет смысл, когда есть несколько типов данных, которые почти не меняются, а над ними нужно делать много разных операций. Желательно ещё, чтобы объекты могли содержать другие объекты — то есть была составная структура.

В качестве примера взял учбный проект и расширил его. Скидки бывают простые — процент или фиксированная сумма, а бывают составные, объединяющие другие скидки. Над ними нужны разные действия: расчёт, описание, проверка корректности, генерация отчётов и прочее.

---

## Исходная версия

### Код

```csharp
public interface IDiscount
{
    Guid Id { get; }
    string Name { get; }

    int CalculateDiscount(int totalAmount);
    string GetDescription();
    bool IsValid();
}

public class PercentDiscount : IDiscount
{
    public Guid Id { get; }
    public string Name { get; }
    public decimal Percent { get; }

    public PercentDiscount(Guid id, string name, decimal percent)
    {
        Id = id;
        Name = name;
        Percent = percent;
    }

    public int CalculateDiscount(int totalAmount)
    {
        return (int)Math.Truncate((totalAmount * Percent) / 100);
    }

    public string GetDescription()
    {
        return $"{Name}: скидка {Percent}%";
    }

    public bool IsValid()
    {
        return Percent >= 0 && Percent <= 100;
    }
}

public class AmountDiscount : IDiscount
{
    public Guid Id { get; }
    public string Name { get; }
    public decimal Amount { get; }

    public AmountDiscount(Guid id, string name, decimal amount)
    {
        Id = id;
        Name = name;
        Amount = amount;
    }

    public int CalculateDiscount(int totalAmount)
    {
        return (int)Amount * 100; // в копейках
    }

    public string GetDescription()
    {
        return $"{Name}: скидка {Amount} руб.";
    }

    public bool IsValid()
    {
        return Amount >= 0;
    }
}

public class CompositeDiscount : IDiscount
{
    public Guid Id { get; }
    public string Name { get; }
    public List<IDiscount> Children { get; }
    public CompositeType Type { get; }

    public CompositeDiscount(Guid id, string name, CompositeType type, List<IDiscount> children)
    {
        Id = id;
        Name = name;
        Type = type;
        Children = children;
    }

    public int CalculateDiscount(int totalAmount)
    {
        if (Children == null || !Children.Any())
            return 0;

        if (Type == CompositeType.Sum)
            return Children.Sum(child => child.CalculateDiscount(totalAmount));
        else
            return Children.Max(child => child.CalculateDiscount(totalAmount));
    }

    public string GetDescription()
    {
        if (Children == null || !Children.Any())
            return $"{Name}: пусто";

        var childDescriptions = Children.Select(c => c.GetDescription());
        var operation = Type == CompositeType.Sum ? "сумма" : "максимум";
        return $"{Name} ({operation}): [{string.Join(", ", childDescriptions)}]";
    }

    public bool IsValid()
    {
        if (Children == null)
            return false;

        return Children.All(c => c.IsValid());
    }
}

public enum CompositeType
{
    Sum,
    Max
}

// Использование
var discount = new CompositeDiscount(
    Guid.NewGuid(),
    "VIP пакет",
    CompositeType.Sum,
    new List<IDiscount>
    {
        new PercentDiscount(Guid.NewGuid(), "Базовая", 10),
        new CompositeDiscount(
            Guid.NewGuid(),
            "Сезонные акции",
            CompositeType.Max,
            new List<IDiscount>
            {
                new PercentDiscount(Guid.NewGuid(), "Лето", 5),
                new AmountDiscount(Guid.NewGuid(), "Фикс", 500)
            }
        )
    }
);

int totalDiscount = discount.CalculateDiscount(10000);
string description = discount.GetDescription();
bool isValid = discount.IsValid();
```

### Проблемы

Когда смотришь на этот код, сразу понятно, куда он ведёт.

Каждая новая операция ломает всё. Добавляешь *GenerateJson()* — меняешь интерфейс и все классы. Добавляешь *CalculateTax()* — снова всё меняешь. Типичное нарушение принципа Open/Closed.

Логика операций разбросана. Чтобы понять, как формируется описание скидки, приходится читать три разных класса. Рекурсивная логика в *CompositeDiscount.GetDescription()* особенно неприятна — не сразу понятно, что происходит.

Каждый класс делает слишком много. *PercentDiscount* одновременно хранит данные, считает, описывает и проверяет себя — это три разные ответственности в одном месте.

И главное — обход составной структуры дублируется. В расчёте используется *Children.Sum(...)*, в описании — *Children.Select(...)*. По сути один и тот же обход реализован по-разному, и добавление новой операции приведёт к новому дублированию.

---

## Рефакторинг с паттерном Visitor

### Код

```csharp
// Базовый класс — только данные
public abstract class Discount
{
    public Guid Id { get; }
    public string Name { get; }

    protected Discount(Guid id, string name)
    {
        Id = id;
        Name = name;
    }

    public abstract T Accept<T>(IDiscountVisitor<T> visitor);
}

public class PercentDiscount : Discount
{
    public decimal Percent { get; }

    public PercentDiscount(Guid id, string name, decimal percent)
        : base(id, name)
    {
        Percent = percent;
    }

    public override T Accept<T>(IDiscountVisitor<T> visitor)
        => visitor.Visit(this);
}

public class AmountDiscount : Discount
{
    public decimal Amount { get; }

    public AmountDiscount(Guid id, string name, decimal amount)
        : base(id, name)
    {
        Amount = amount;
    }

    public override T Accept<T>(IDiscountVisitor<T> visitor)
        => visitor.Visit(this);
}

public class CompositeDiscount : Discount
{
    public List<Discount> Children { get; }
    public CompositeType Type { get; }

    public CompositeDiscount(Guid id, string name, CompositeType type, List<Discount> children)
        : base(id, name)
    {
        Type = type;
        Children = children ?? new List<Discount>();
    }

    public override T Accept<T>(IDiscountVisitor<T> visitor)
        => visitor.Visit(this);
}

// Интерфейс для операций
public interface IDiscountVisitor<T>
{
    T Visit(PercentDiscount discount);
    T Visit(AmountDiscount discount);
    T Visit(CompositeDiscount discount);
}

// Операция 1: расчёт
public class DiscountCalculationVisitor : IDiscountVisitor<int>
{
    private readonly int _totalAmount;

    public DiscountCalculationVisitor(int totalAmount)
    {
        _totalAmount = totalAmount;
    }

    public int Visit(PercentDiscount d)
        => (int)Math.Truncate((_totalAmount * d.Percent) / 100);

    public int Visit(AmountDiscount d)
        => (int)d.Amount * 100;

    public int Visit(CompositeDiscount d)
    {
        var results = d.Children.Select(c => c.Accept(this));

        return d.Type == CompositeType.Sum
            ? results.Sum()
            : results.DefaultIfEmpty(0).Max();
    }
}

// Операция 2: описание
public class DiscountDescriptionVisitor : IDiscountVisitor<string>
{
    public string Visit(PercentDiscount d)
        => $"{d.Name}: скидка {d.Percent}%";

    public string Visit(AmountDiscount d)
        => $"{d.Name}: скидка {d.Amount} руб.";

    public string Visit(CompositeDiscount d)
    {
        var children = d.Children.Select(c => c.Accept(this));
        var operation = d.Type == CompositeType.Sum ? "сумма" : "максимум";
        return $"{d.Name} ({operation}): [{string.Join(", ", children)}]";
    }
}

// Операция 3: валидация
public class DiscountValidationVisitor : IDiscountVisitor<bool>
{
    public bool Visit(PercentDiscount d)
        => d.Percent >= 0 && d.Percent <= 100;

    public bool Visit(AmountDiscount d)
        => d.Amount >= 0;

    public bool Visit(CompositeDiscount d)
        => d.Children.All(c => c.Accept(this));
}

// Использование
var discount = new CompositeDiscount(
    Guid.NewGuid(),
    "VIP пакет",
    CompositeType.Sum,
    new List<Discount>
    {
        new PercentDiscount(Guid.NewGuid(), "Базовая", 10),
        new CompositeDiscount(
            Guid.NewGuid(),
            "Сезонные акции",
            CompositeType.Max,
            new List<Discount>
            {
                new PercentDiscount(Guid.NewGuid(), "Лето", 5),
                new AmountDiscount(Guid.NewGuid(), "Фикс", 500)
            }
        )
    }
);

int totalDiscount = discount.Accept(new DiscountCalculationVisitor(10000));
string description = discount.Accept(new DiscountDescriptionVisitor());
bool isValid = discount.Accept(new DiscountValidationVisitor());
```

### Что изменилось

Классы скидок теперь чисто хранят данные и метод *Accept*. Никакой бизнес-логики внутри.

Каждая операция вынесена в отдельный класс — visitor. Вся логика расчёта в *DiscountCalculationVisitor*, описание в *DiscountDescriptionVisitor*, валидация в *DiscountValidationVisitor*. Теперь не нужно прыгать между файлами, чтобы понять, как работает одна операция.

Рекурсивный обход читается естественно: *d.Children.Select(c => c.Accept(this))* — берём результаты от детей и комбинируем. Без лишних стеков и состояния.

Добавить новую операцию просто: создаёшь visitor, реализуешь методы для существующих типов, и всё готово. Классы скидок не трогаешь.

---

## Сравнение

### Что стало лучше

Добавлять новые операции стало легко: JSON, SQL, отчёты, проверка — всё через новые visitor. Классы скидок даже не знают о существовании этих операций.

Тестировать проще, пример:

```csharp
[Fact]
public void CalculatesPercentDiscount()
{
    var discount = new PercentDiscount(Guid.NewGuid(), "Test", 10);
    var visitor = new DiscountCalculationVisitor(10000);

    int result = discount.Accept(visitor);

    Assert.Equal(1000, result);
}
```

Всё понятно с первого взгляда.

### Что стало хуже

Добавление нового типа скидки теперь дороже. Например, *ConditionalDiscount*: нужно создать класс, добавить *Visit* в интерфейс, реализовать во всех visitor'ах. Пять visitor'ов — пять мест для изменений.

Появился boilerplate: каждый класс пишет *Accept*, каждый visitor — три метода *Visit*.

Для новичка double dispatch (*discount.Accept(visitor)* вызывает *visitor.Visit(this)*) может быть непривычен.

### Когда это оправдано

Visitor подходит здесь, потому что структура стабильна: три типа скидок (процент, фиксированная сумма, составная) — фундаментальные блоки, новые редко добавляются.

Операций много: расчёт, описание, валидация, отчёты, экспорт, логирование — и они будут добавляться.

Плюс есть составная структура: *CompositeDiscount* содержит другие скидки. Это главное преимущество Visitor: логика обхода централизована, а не дублируется в каждой операции.

Если бы типов скидок было много и они часто менялись, а операций мало — Visitor был бы невыгоден.

---

## Итог

Главное, что я вынес: Visitor — это не просто способ вынести логику из классов. Он про добавление операций над стабильной структурой.

Если структура стабильна, операций много и они регулярно появляются — Visitor упрощает жизнь. Если всё наоборот — только усложняет.

Ещё один важный момент: Visitor — это прежде всего про обход составной структуры. В исходном коде каждая операция обходила дочерние элементы по-своему. С Visitor обход стал явным: *child.Accept(this)* — сразу видно, что происходит.

Паттерны — инструменты для конкретных задач. Visitor — для стабильных объектов с множеством операций. Strategy — для взаимозаменяемых алгоритмов. Factory — для создания разных типов объектов. Понимание задачи важнее, чем простое следование паттерну.