# Отчёт: Как правильно писать тесты

## Введение

Я перечитал материал про то, как правильно писать тесты, и решил проверить его не в теории, а на практике — на реальных тестах из проекта. Хотелось понять простую вещь: мы вообще так работаем или только говорим, что работаем?

Главная мысль материала одновременно простая и неприятная: тесты всегда должны быть зелёными. Не «в конце», не «после того как допишу всё», а всегда. Если тест упал — откатываешься к последней рабочей ревизии и пробуешь снова. Без накопления изменений и без попыток «сейчас быстренько всё починю».

Из этого вытекают три правила:

Сначала добавить тест — пусть даже с фейковой реализацией.

Постепенно заменять фейк на настоящую логику.

Любое усложнение делать через вспомогательные функции.

Звучит разумно. Но когда смотришь на реальные тесты в проекте, становится видно, как легко от этого процесса отойти.

---

# Итерация 1: Тесты DishComparer с дублированием setup-кода

## Исходный код

```csharp
public class DishDtoComparerTests
{
    private readonly IMapper _mapper;

    public DishDtoComparerTests()
    {
        var configuration = new MapperConfiguration(cfg =>
        {
            cfg.CreateMap<DishDto, Dish>();
        });

        _mapper = configuration.CreateMapper();
    }

    [Fact]
    public void HasChanges_IdenticalArrays_ReturnsFalse()
    {
        // Arrange
        var oldDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 11, Quantity = 1 },
                new DishDto { Id = 12, Quantity = 3 }
            }},
            new DishDto { Id = 2, Quantity = 5 }
        };

        var newDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 11, Quantity = 1 },
                new DishDto { Id = 12, Quantity = 3 }
            }},
            new DishDto { Id = 2, Quantity = 5 }
        };

        var oldDishes = _mapper.Map<Dish[]>(oldDishesDto);
        var newDishes = _mapper.Map<Dish[]>(newDishesDto);

        // Act
        var result = DishComparer.HasChanges(oldDishes, newDishes);

        // Assert
        Assert.False(result);
    }

    [Fact]
    public void HasChanges_SameCompositionDifferentOrder_ReturnsFalse()
    {
        // Arrange
        var oldDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 11, Quantity = 1 },
                new DishDto { Id = 12, Quantity = 3 }
            }},
            new DishDto { Id = 2, Quantity = 5 }
        };

        var newDishesDto = new DishDto[]
        {
            new DishDto { Id = 2, Quantity = 5 },
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 12, Quantity = 3 },
                new DishDto { Id = 11, Quantity = 1 }
            }}
        };

        var oldDishes = _mapper.Map<Dish[]>(oldDishesDto);
        var newDishes = _mapper.Map<Dish[]>(newDishesDto);

        // Act
        var result = DishComparer.HasChanges(oldDishes, newDishes);

        // Assert
        Assert.False(result);
    }

    [Fact]
    public void HasChanges_DifferentComposition_ReturnsTrue()
    {
        // Arrange
        var oldDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 11, Quantity = 1 },
                new DishDto { Id = 12, Quantity = 3 }
            }},
            new DishDto { Id = 2, Quantity = 5 }
        };

        var newDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2, Dishes = new List<DishDto>
            {
                new DishDto { Id = 11, Quantity = 2 }, // Different quantity
                new DishDto { Id = 12, Quantity = 3 }
            }},
            new DishDto { Id = 2, Quantity = 5 }
        };

        var oldDishes = _mapper.Map<Dish[]>(oldDishesDto);
        var newDishes = _mapper.Map<Dish[]>(newDishesDto);

        // Act
        var result = DishComparer.HasChanges(oldDishes, newDishes);

        // Assert
        Assert.True(result);
    }

    [Fact]
    public void HasChanges_NullArrays_ReturnsTrue()
    {
        // Arrange
        DishDto[] oldDishDtoes = null;
        var newDishesDto = new DishDto[]
        {
            new DishDto { Id = 1, Quantity = 2 }
        };

        var newDishes = _mapper.Map<Dish[]>(newDishesDto);

        // Act
        var result = DishComparer.HasChanges(null, newDishes);

        // Assert
        Assert.True(result);
    }

    [Fact]
    public void HasChanges_BothNullArrays_ReturnsFalse()
    {
        // Act
        var result = DishComparer.HasChanges(null, null);

        // Assert
        Assert.True(result);
    }
}
```

## Проблемы с точки зрения «маленьких шагов»

Когда смотришь на эти тесты подряд, первое ощущение — они просто тяжёлые. Не по логике, а по объёму.

Каждый тест начинается с большого Arrange: массивы, вложенные *DishDto*, AutoMapper, повторяющиеся конструкции. Чтобы понять, что именно проверяется, приходится продираться через детали, которые к сути теста не относятся. Если бы я писал это маленькими шагами, я бы начал с минимального случая и усложнял постепенно.

Вторая проблема — явный баг, который легко пропустить. Тест называется *HasChanges_BothNullArrays_ReturnsFalse*, а внутри стоит *Assert.True*. Кто-то из них врёт. Скорее всего — ассерт. Это классическая ошибка копипаста: скопировали предыдущий тест, поменяли входные данные и забыли поменять ожидание.

Важно не то, что баг есть, а почему он вообще появился. Скорее всего, тесты писались пачкой и запускались уже потом. Если бы тест запускался сразу после написания, он бы просто не стал зелёным.

Третья проблема — слишком много кода между моментом «написал тест» и моментом «увидел зелёный». Видно, что разработчик сначала написал все тесты, а потом начал чинить реализацию. Материал как раз от этого и предостерегает.

И наконец — отсутствие постепенности. Первый же тест проверяет сложную структуру с вложенностью и игнорированием порядка. Гораздо логичнее было бы начать с одного элемента без вложенных списков и шаг за шагом добавлять сложность.

## Как это можно было писать маленькими шагами

Я попробовал применить три правила из материала.

**Правило 1: добавить тест, чтобы он прошёл**

Начинаю с самого простого случая — пустые массивы. Даже с фейковой реализацией.

```csharp
// Шаг 1: Простейший тест - пустые массивы
[Fact]
public void HasChanges_EmptyArrays_ReturnsFalse()
{
    // Arrange
    var oldDishes = Array.Empty<Dish>();
    var newDishes = Array.Empty<Dish>();

    // Act
    var result = DishComparer.HasChanges(oldDishes, newDishes);

    // Assert
    Assert.False(result);
}

// Реализация (фейк, но тест зелёный):
public static class DishComparer
{
    public static bool HasChanges(Dish[] oldDishes, Dish[] newDishes)
    {
        return false; // Фейк: всегда возвращаем false
    }
}
```

Тест зелёный — коммит. В этот момент важен не «идеальный код», а то, что система в рабочем состоянии.

**Правило 2: постепенно заменять фейк на настоящую реализацию**

Следующий шаг — один элемент без вложенности.

```csharp
// Шаг 2: Простой случай - один элемент, одинаковые
[Fact]
public void HasChanges_SingleIdenticalDish_ReturnsFalse()
{
    // Arrange
    var oldDishes = new[] { new Dish { Id = 1, Quantity = 2 } };
    var newDishes = new[] { new Dish { Id = 1, Quantity = 2 } };

    // Act
    var result = DishComparer.HasChanges(oldDishes, newDishes);

    // Assert
    Assert.False(result);
}

// Реализация (чуть менее фейковая):
public static bool HasChanges(Dish[] oldDishes, Dish[] newDishes)
{
    if (oldDishes.Length != newDishes.Length)
        return true;
        
    return false; // Пока упрощённо
}
```

Реализация всё ещё упрощённая, но тесты зелёные. Коммит.

Добавляю тест, который действительно что-то ломает — разное количество.

```csharp
// Шаг 3: Проверка что разные элементы детектируются
[Fact]
public void HasChanges_DifferentQuantity_ReturnsTrue()
{
    // Arrange
    var oldDishes = new[] { new Dish { Id = 1, Quantity = 2 } };
    var newDishes = new[] { new Dish { Id = 1, Quantity = 3 } };

    // Act
    var result = DishComparer.HasChanges(oldDishes, newDishes);

    // Assert
    Assert.True(result);
}

// Реализация (добавляем реальную логику):
public static bool HasChanges(Dish[] oldDishes, Dish[] newDishes)
{
    if (oldDishes.Length != newDishes.Length)
        return true;
    
    for (int i = 0; i < oldDishes.Length; i++)
    {
        if (oldDishes[i].Id != newDishes[i].Id || 
            oldDishes[i].Quantity != newDishes[i].Quantity)
            return true;
    }
    
    return false;
}
```

Теперь логика стала реальной, но всё ещё простой. Тесты зелёные — ещё один коммит.

**Правило 3: делать сложные изменения простыми**

Вместо того чтобы сразу городить сложную логику, я выношу нормализацию порядка в отдельную функцию.

```csharp
// Шаг 4: Вспомогательная функция для нормализации порядка
private static Dish[] NormalizeOrder(Dish[] dishes)
{
    return dishes.OrderBy(d => d.Id).ToArray();
}

// Теперь могу добавить тест на порядок
[Fact]
public void HasChanges_DifferentOrder_ReturnsFalse()
{
    // Arrange
    var oldDishes = new[] 
    { 
        new Dish { Id = 1, Quantity = 2 },
        new Dish { Id = 2, Quantity = 5 }
    };
    var newDishes = new[] 
    { 
        new Dish { Id = 2, Quantity = 5 },
        new Dish { Id = 1, Quantity = 2 }
    };

    // Act
    var result = DishComparer.HasChanges(oldDishes, newDishes);

    // Assert
    Assert.False(result);
}

// Реализация (используем вспомогательную функцию):
public static bool HasChanges(Dish[] oldDishes, Dish[] newDishes)
{
    oldDishes = NormalizeOrder(oldDishes);
    newDishes = NormalizeOrder(newDishes);
    
    if (oldDishes.Length != newDishes.Length)
        return true;
    
    for (int i = 0; i < oldDishes.Length; i++)
    {
        if (oldDishes[i].Id != newDishes[i].Id || 
            oldDishes[i].Quantity != newDishes[i].Quantity)
            return true;
    }
    
    return false;
}
```

После этого добавление теста на порядок элементов становится тривиальным. Тест зелёный — коммит. Дальше можно так же аккуратно добавлять вложенные списки, null-проверки и всё остальное.

## Выводы по первой итерации

В исходном варианте разработчик написал сразу 7 тестов со сложными Arrange-секциями, потом запустил их и начал чинить. В таком подходе легко:

накопить много изменений,

допустить ошибку в ассерте,

потерять понимание, что именно сломалось.

Подход маленькими шагами выглядит скучнее, но даёт контроль. Каждый тест — отдельный шаг, каждый шаг — зелёный, каждый шаг легко откатить.

---

# Итерация 2: Тесты GuestBirthday с избыточной детализацией

## Исходный код

```csharp
public class GuestBirthdayTests
{
    protected readonly IMapper _mapper;

    public GuestBirthdayTests()
    {
        var configuration = new MapperConfiguration(cfg =>
        {
            cfg.CreateMap<GuestDto, Guest>();
            cfg.CreateMap<OrderDto, Order>();
            cfg.CreateMap<DishDto, Dish>();
            cfg.CreateMap<DiscountedDishDto, DiscountedDish>();
            cfg.CreateMap<CalculationDto, Calculation>();
        });

        _mapper = configuration.CreateMapper();
    }

    /// <summary>
    /// Проверяет, что условие не срабатывает, когда день рождения гостя не указан (null)
    /// </summary>
    [Trait("Условия", "День рождения Гостя")]
    [Fact(DisplayName = "Условие не срабатывает, когда день рождения гостя не указан (null)")]
    public void IsMatch_WhenGuestBirthdayIsNull_ReturnsFalse()
    {
        // Arrange
        var condition = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };
        var calculationDto = new CalculationDto
        {
            Guest = new GuestDto { Birthday = null },
            CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
        };
        var calculation = _mapper.Map<Calculation>(calculationDto);

        // Act
        var result = condition.IsMatch(calculation);

        // Assert
        Assert.False(result);
    }

    /// <summary>
    /// Проверяет, что условие срабатывает, когда день рождения находится в пределах диапазона
    /// </summary>
    [Trait("Условия", "День рождения Гостя")]
    [Fact(DisplayName = "Условие срабатывает, когда день рождения находится в пределах диапазона")]
    public void IsMatch_WhenBirthdayIsWithinRange_ReturnsTrue()
    {
        // Arrange
        var condition = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };
        var calculationDto = new CalculationDto
        {
            Guest = new GuestDto { Birthday = new DateOnly(2023, 1, 10) },
            CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
        };
        var calculation = _mapper.Map<Calculation>(calculationDto);

        // Act
        var result = condition.IsMatch(calculation);

        // Assert
        Assert.True(result);
    }

    // ... ещё 9 похожих тестов с boundary cases ...

    /// <summary>
    /// Проверяет, что условие корректно работает с временной зоной DateTimeOffset
    /// </summary>
    [Trait("Условия", "День рождения Гостя")]
    [Fact(DisplayName = "Условие корректно работает с временной зоной DateTimeOffset")]
    public void IsMatch_WithDifferentTimeZone_ReturnsCorrectResult()
    {
        // Arrange
        var condition = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };
        var calculationDto = new CalculationDto
        {
            Guest = new GuestDto { Birthday = new DateOnly(2023, 1, 10) },
            CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.FromHours(3)) // UTC+3
        };
        var calculation = _mapper.Map<Calculation>(calculationDto);

        // Act
        var result = condition.IsMatch(calculation);

        // Assert
        Assert.True(result);
    }
}
```

## Проблемы с точки зрения маленьких шагов

Здесь другая крайность. Тесты выглядят очень профессионально: XML-документация, DisplayName, Trait, покрытие всех boundary cases. Но за этим теряется процесс.

Во-первых, почти наверняка все 11 тестов писались за один присест. Разработчик сначала выписал все edge cases, потом реализовал всё сразу. Материал предлагает обратное: не держать в голове десяток изменений одновременно.

Во-вторых, тест с таймзоной выглядит как поздняя мысль: «а что если timezone другая?». Если бы работа шла маленькими шагами, этот случай появился бы раньше, на этапе базовой логики.

В-третьих, каждый тест повторяет один и тот же шаблон: создание *CalculationDto*, маппинг, вызов *IsMatch*. Это явный кандидат на вспомогательную функцию.

И наконец, здесь вообще нет фейковой реализации. Сначала была написана полноценная логика, а уже потом — все тесты. Это больше «разработка с тестами», чем TDD.

## Как это можно было писать маленькими шагами

**Правило 1: минимальный тест и фейк**

Начинаю с самого базового сценария — день рождения совпадает с датой заказа.

```csharp
// Шаг 1: Простейший тест - день рождения совпадает с датой заказа
[Fact]
public void IsMatch_BirthdayOnOrderDate_ReturnsTrue()
{
    // Arrange
    var condition = new GuestBirthday();
    var calculation = new Calculation
    {
        Guest = new Guest { Birthday = new DateOnly(2023, 1, 10) },
        CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
    };

    // Act
    var result = condition.IsMatch(calculation);

    // Assert
    Assert.True(result);
}

// Реализация (фейк):
public bool IsMatch(Calculation calculation)
{
    return true; // Фейк: всегда true
}
```

Фейк, тест зелёный, коммит.

**Правило 2: постепенное усложнение**

Добавляю тест на null.

```csharp
// Шаг 2: Тест на null birthday
[Fact]
public void IsMatch_BirthdayIsNull_ReturnsFalse()
{
    // Arrange
    var condition = new GuestBirthday();
    var calculation = new Calculation
    {
        Guest = new Guest { Birthday = null },
        CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
    };

    // Act
    var result = condition.IsMatch(calculation);

    // Assert
    Assert.False(result);
}

// Реализация (чуть менее фейковая):
public bool IsMatch(Calculation calculation)
{
    if (calculation.Guest.Birthday == null)
        return false;
        
    return true; // Пока упрощённо
}
```

Потом — простой диапазон.

```csharp
// Шаг 3: Проверка простого диапазона
[Fact]
public void IsMatch_BirthdayTomorrow_ReturnsTrue()
{
    // Arrange
    var condition = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };
    var calculation = new Calculation
    {
        Guest = new Guest { Birthday = new DateOnly(2023, 1, 11) },
        CreateDate = new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
    };

    // Act
    var result = condition.IsMatch(calculation);

    // Assert
    Assert.True(result);
}

// Реализация (добавляем логику диапазона):
public bool IsMatch(Calculation calculation)
{
    if (calculation.Guest.Birthday == null)
        return false;
    
    var birthday = calculation.Guest.Birthday.Value;
    var orderDate = DateOnly.FromDateTime(calculation.CreateDate.Date);
    
    var daysDiff = (birthday.ToDateTime(TimeOnly.MinValue) - orderDate.ToDateTime(TimeOnly.MinValue)).Days;
    
    return daysDiff >= -DaysBefore && daysDiff <= DaysAfter;
}
```

Логика появляется постепенно, без скачков. Каждый шаг зелёный.

**Правило 3: вспомогательные функции**

Чтобы не копировать одно и то же, выношу создание Calculation в builder.

```csharp
// Вспомогательная функция для создания тестовых данных
private Calculation CreateCalculation(DateOnly? birthday, DateTimeOffset orderDate)
{
    return new Calculation
    {
        Guest = new Guest { Birthday = birthday },
        CreateDate = orderDate
    };
}

// Теперь тесты короче
[Fact]
public void IsMatch_BirthdayYesterday_ReturnsTrue()
{
    // Arrange
    var condition = new GuestBirthday { DaysBefore = 5, DaysAfter = 5 };
    var calculation = CreateCalculation(
        birthday: new DateOnly(2023, 1, 9),
        orderDate: new DateTimeOffset(2023, 1, 10, 0, 0, 0, TimeSpan.Zero)
    );

    // Act
    var result = condition.IsMatch(calculation);

    // Assert
    Assert.True(result);
}
```

Теперь добавлять boundary cases легко и безопасно — каждый новый тест маленький и изолированный.

## Выводы по второй итерации

Исходный подход даёт ощущение завершённости, но плохо показывает путь. Непонятно, что было сделано сначала, а что появилось позже. При ошибке сложно понять, где именно процесс дал сбой.

Подход маленькими шагами оставляет историю: каждый коммит — понятное состояние системы.

---

## Общая рефлексия

Главное, что я вынес из этого материала: процесс важнее результата. Можно написать хорошие тесты плохим процессом, но это дорого и рискованно.

TDD — это не «наличие тестов», а цикл: тест → фейк → зелёный → коммит → усложнение. Фейк — не обман, а инструмент, который позволяет двигаться, не теряя контроля.

Маленькие шаги снимают когнитивную нагрузку. Когда тесты всегда зелёные, не нужно держать в голове «а вдруг что-то сломалось». Я знаю, что система работает, и могу спокойно делать следующий шаг.

Постоянные коммиты — это не overhead, а страховка. Потерять пять минут работы не страшно. Потерять час — уже больно.

После этого задания стало ясно: тесты — это не документация и не формальность. Это инструмент мышления и способ двигаться вперёд, не теряя уверенности.
