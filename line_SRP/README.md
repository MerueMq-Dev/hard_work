# Отчёт: SRP на уровне строки кода

## Введение

Честно говоря, когда увидел задание «найди нарушения SRP в одной строке», первая реакция была «ну, это же просто длинные строки». Оказалось — нет. Длина следствие, а причина в другом: одна строка делает несколько вещей одновременно, и читатель вынужден разбирать их все сразу, не имея возможности остановиться на чём-то одном.

Прошёлся по живому коду и нашёл семь конкретных мест. Некоторые знакомы по прошлым итерациям, некоторые — новые. Общий паттерн везде один: автор знал, что делает, но не дал читателю ни одной подсказки — ни имени переменной, ни разбивки. Всё слито в одну строку.

---

## Пример 1. Проверка трёх несвязанных свойств в одном условии

В *DishComparer.HasChanges* одна строка проверяет сразу три изменения блюда: флаг исключения скидки, идентификатор и количество. Это три разных бизнес-правила, склеенных через *||*. Если условие сработало — непонятно, что именно изменилось.

### Было

```csharp
if (oldItem.IsDiscountExcluded != newItem.IsDiscountExcluded || oldItem.Id != newItem.Id || oldItem.Quantity != newItem.Quantity)
    return true;
```

### Стало

```csharp
// Каждая проверка читается как отдельное правило — и в логах, и при отладке
// понятно, что именно изменилось
bool dishIdentityChanged   = oldItem.Id != newItem.Id;
bool quantityChanged       = oldItem.Quantity != newItem.Quantity;
bool discountExclusionChanged = oldItem.IsDiscountExcluded != newItem.IsDiscountExcluded;

if (dishIdentityChanged || quantityChanged || discountExclusionChanged)
    return true;
```

Что улучшили: три проверки получили имена. Теперь при добавлении четвёртой проверки (например, по цене) не надо разбирать, что уже есть в строке — просто добавляем новую переменную.

---

## Пример 2. Конвертация строки и парсинг в одном вызове

В *Mapper.Map* для *Group* — строки из внешнего API приходят как *string*, их нужно конвертировать в *long*. Это делается прямо в *Select*, без промежуточного шага. Если *long.Parse* упадёт — стек трейс укажет на лямбду внутри *Select*, а не на конкретную строку, вызвавшую проблему.

### Было

```csharp
return new Group(rest.CorporationId, rest.Id, long.Parse(x.Id), x.Name,
    x.Ingredients.Select(i => long.Parse(i)).ToList());
```

### Стало

```csharp
// Парсинг вынесен явно — если упадёт, понятно где и на каком значении
long groupId = long.Parse(x.Id);
List<long> ingredientIds = x.Ingredients
    .Select(i => long.Parse(i))
    .ToList();

return new Group(rest.CorporationId, rest.Id, groupId, x.Name, ingredientIds);
```

Что улучшили: парсинг отделён от конструирования объекта. Ошибка при некорректном *Id* теперь видна без распутывания вложенных вызовов, а *ingredientIds* можно проверить в дебаггере до того, как он уйдёт в конструктор *Group*.

---

## Пример 3. Три уровня вложенных *Select* в одной строке

В *Calculation.InitDiscountedDishes* — преобразование иерархии блюд из *Dish* в *DiscountedDish*. Три уровня вложенности (блюдо → ингредиент → модификатор) написаны одной непрерывной строкой. Добавить четвёртый уровень или изменить маппинг одного уровня — значит разобрать всю конструкцию целиком.

### Было

```csharp
DiscountedDishes = OriginalDishes.Select(x => new DiscountedDish(x.LineItemId!.Value, x.Id, x.Name, x.Quantity, x.Price, x.Amount, x.Restrictions,
    (x.Dishes != null) ? x.Dishes.Select(i => new DiscountedDish(i.LineItemId!.Value, i.Id, i.Name, i.Quantity, i.Price, i.Amount, i.Restrictions,
        (i.Dishes != null) ? i.Dishes.Select(m => new DiscountedDish(m.LineItemId!.Value, m.Id, m.Name, m.Quantity, m.Price, m.Amount, m.Restrictions,
            isDiscountExcluded: m.IsDiscountExcluded)).ToList() : null,
        isDiscountExcluded: i.IsDiscountExcluded)).ToList() : null,
    isDiscountExcluded: x.IsDiscountExcluded)).ToList();
```

### Стало

```csharp
// Каждый уровень иерархии — отдельный метод с понятным именем.
// Изменить маппинг модификатора можно не трогая маппинг блюда.
DiscountedDishes = OriginalDishes.Select(MapToDiscountedDish).ToList();

DiscountedDish MapToDiscountedDish(Dish x) =>
    new DiscountedDish(x.LineItemId!.Value, x.Id, x.Name, x.Quantity, x.Price, x.Amount,
        x.Restrictions,
        dishes: x.Dishes?.Select(MapToDiscountedIngredient).ToList(),
        isDiscountExcluded: x.IsDiscountExcluded);

DiscountedDish MapToDiscountedIngredient(Dish i) =>
    new DiscountedDish(i.LineItemId!.Value, i.Id, i.Name, i.Quantity, i.Price, i.Amount,
        i.Restrictions,
        dishes: i.Dishes?.Select(MapToDiscountedModificator).ToList(),
        isDiscountExcluded: i.IsDiscountExcluded);

DiscountedDish MapToDiscountedModificator(Dish m) =>
    new DiscountedDish(m.LineItemId!.Value, m.Id, m.Name, m.Quantity, m.Price, m.Amount,
        m.Restrictions,
        isDiscountExcluded: m.IsDiscountExcluded);
```

Что улучшили: каждый уровень иерархии — отдельная ответственность с именем. Тест на маппинг модификатора теперь не тащит за собой всю цепочку. Добавить новый уровень или изменить поля одного уровня — операция в одном месте.

---

## Пример 4. Создание *DishTaxes* с вложенным *Select* прямо в аргументе конструктора

В *Mapper.Map* для *Product* — условный оператор, внутри которого сразу же создаётся *DishTaxes*, внутри которого — *Select* по *Taxes*. Три логических операции в одной строке-аргументе: проверка на *null*, создание объекта, преобразование коллекции.

### Было

```csharp
(x.DishTaxes != null)
    ? new DishTaxes(x.DishTaxes.GroupName, x.DishTaxes.Taxes.Select(t => new Tax(t.TaxName, t.RateName, t.Rate)).ToList())
    : null,
```

### Стало

```csharp
// Каждое действие — отдельная строка с понятной ответственностью:
// 1. Проверка наличия налогов
// 2. Преобразование каждого налога
// 3. Создание объекта-контейнера
DishTaxes dishTaxes = null;
if (x.DishTaxes != null)
{
    var taxes = x.DishTaxes.Taxes
        .Select(t => new Tax(t.TaxName, t.RateName, t.Rate))
        .ToList();
    dishTaxes = new DishTaxes(x.DishTaxes.GroupName, taxes);
}
```

Что улучшили: *taxes* теперь именованная переменная — её можно проверить в отладчике. Если придёт *Taxes == null*, *NullReferenceException* будет на конкретной строке, а не внутри однострочного тернарника.

---

## Пример 5. Вычисление лимита скидки с несколькими преобразованиями в одном *Select*

В *Calculation.InitDiscountedDishes* — расчёт максимально возможной скидки по блюдам. В одном *Select*-выражении: умножение количества на разницу цены и минимальной цены, плюс каст к *int*, плюс нулевой коалесцинг. Формула бизнес-правила полностью скрыта за синтаксическим шумом.

### Было

```csharp
TotalLimitDiscount = Calculator.CalculateTotal(DiscountedDishes, x => (int)(x.Quantity * (x.Price - (x.Restrictions.MinPrice ?? 0))));
```

### Стало

```csharp
// Формула читается как бизнес-правило:
// «максимальная скидка на блюдо — это количество умножить на разницу между ценой и минимальной ценой»
TotalLimitDiscount = Calculator.CalculateTotal(DiscountedDishes, MaxPossibleDiscountPerDish);

long MaxPossibleDiscountPerDish(DiscountedDish x)
{
    int minPrice = x.Restrictions.MinPrice ?? 0;
    int discountableAmount = x.Price - minPrice;
    return (long)(x.Quantity * discountableAmount);
}
```

Что улучшили: формула получила имя *MaxPossibleDiscountPerDish* — теперь читается как намерение, а не как выражение. *minPrice* и *discountableAmount* — это именованные шаги, каждый из которых можно проверить. Каст к *long* вынесен явно и не теряется внутри выражения.

---

## Пример 6. Маппинг трёх уровней блюд в теле *SetDishes* одной строкой

В *Calculations.RequestHandler.Handle* — вызов *SetDishes* с тремя уровнями вложенных *Select* прямо в аргументе. Это хуже примера 3 тем, что здесь ещё и *Restrictions* создаётся inline — итого четыре разных действия в одном выражении-аргументе.

### Было

```csharp
calculation.SetDishes(request.Order.Dishes.Select(x => new Dish(x.Id, x.Name, x.Quantity, x.Price, x.Amount, new Restrictions(x.MinPrice ?? 0, x.MaxPrice ?? x.Price), lineItemId: x.LineItemId,                    
    dishes: (x.Dishes != null) ? x.Dishes.Select(i => new Dish(i.Id, i.Name, i.Quantity, i.Price, i.Amount, new Restrictions(i.MinPrice ?? 0, i.MaxPrice ?? i.Price),
    dishes: (i.Dishes != null) ? i.Dishes.Select(m => new Dish(m.Id, m.Name, m.Quantity, m.Price, m.Amount, new Restrictions(m.MinPrice ?? 0, m.MaxPrice ?? m.Price), lineItemId: m.LineItemId)).ToList() : null, lineItemId: i.LineItemId)).ToList() : null,
    isDiscountExcluded: x.IsDiscountExcluded ?? false
    )),
    request.Promocode);
```

### Стало

```csharp
// Маппинг каждого уровня — отдельный метод.
// SetDishes получает готовую коллекцию, а не инструкцию по её сборке.
var dishes = request.Order.Dishes.Select(MapDish);
calculation.SetDishes(dishes, request.Promocode);

Dish MapDish(CalcDishDto x) =>
    new Dish(x.Id, x.Name, x.Quantity, x.Price, x.Amount,
        new Restrictions(x.MinPrice ?? 0, x.MaxPrice ?? x.Price),
        lineItemId: x.LineItemId,
        dishes: x.Dishes?.Select(MapIngredient).ToList(),
        isDiscountExcluded: x.IsDiscountExcluded ?? false);

Dish MapIngredient(CalcIngredientDto i) =>
    new Dish(i.Id, i.Name, i.Quantity, i.Price, i.Amount,
        new Restrictions(i.MinPrice ?? 0, i.MaxPrice ?? i.Price),
        lineItemId: i.LineItemId,
        dishes: i.Dishes?.Select(MapModificator).ToList());

Dish MapModificator(CalcModificatorDto m) =>
    new Dish(m.Id, m.Name, m.Quantity, m.Price, m.Amount,
        new Restrictions(m.MinPrice ?? 0, m.MaxPrice ?? m.Price),
        lineItemId: m.LineItemId);
```

Что улучшили: *SetDishes* теперь получает данные, а не рецепт их приготовления. Маппинг каждого уровня изолирован — тест на маппинг модификатора не затрагивает блюдо. Правило «если MinPrice не передан — берём 0» теперь читается явно в каждом методе, а не угадывается из *??*.

---

## Пример 7. Вычисление суммы со скидкой и без скидки в одной строке *TotalAmountWithDiscount*

В *InitDiscountedDishes* — *TotalAmountWithDiscount* считается как *CalculateTotal* с лямбдой, которая вычитает скидку прямо внутри себя. В той же строке — скрытое бизнес-правило «итог = сумма минус скидка». При этом строчкой выше *TotalAmount* считается иначе — через просто *x.Amount*. Несогласованность читается как ошибка, хотя это намерение.

### Было

```csharp
TotalAmount = Calculator.CalculateTotal(OriginalDishes, x => x.Amount);
TotalAmountWithDiscount = Calculator.CalculateTotal(DiscountedDishes, x => (x.Amount - x.Discount));
```

### Стало

```csharp
// Правило «сумма с учётом скидки» — это свойство блюда, а не формула в аргументе.
// FullAmount уже существует в DiscountedDish — используем его явно.
TotalAmount = Calculator.CalculateTotal(OriginalDishes, x => x.Amount);
TotalAmountWithDiscount = Calculator.CalculateTotal(DiscountedDishes, x => x.FullAmount);

// Для справки — FullAmount уже определён в DiscountedDish:
// public long FullAmount => Amount - Discount;
```

Что улучшили: формула *Amount - Discount* теперь не дублируется в каждом *Select* — она живёт в одном месте как свойство *FullAmount*. Строки стали симметричными и читаются одинаково: обе берут одно свойство блюда, разница только в коллекции. Намерение сразу видно.

---

## Общая рефлексия

Честно говоря, в начале я думал, что SRP на уровне строки — это просто «не пиши длинные строки». Оказалось, что нет. Длина — симптом, а болезнь — это когда одна строка одновременно проверяет условие, создаёт объект и трансформирует коллекцию. Читатель вынужден держать всё это в голове сразу.

И вот что интересно: до этого задания я вообще не думал об SRP на уровне строки. SRP — это про классы, ну максимум про методы. Но оказывается, принцип работает на любом уровне детализации, вплоть до одной физической строки. Это немного меняет то, как смотришь на код при ревью: теперь вопрос не только «правильно ли разбиты классы», но и «что делает эта конкретная строка — одну вещь или несколько».

Главное, что вынес из семи примеров: у каждого действия должно быть имя. Не *(x.Amount - x.Discount)* в аргументе — а *x.FullAmount*. Не *long.Parse(x.Id)* внутри конструктора — а *long groupId = long.Parse(x.Id)* перед ним. Не три вложенных *Select* — а три метода с именами уровней.

Ещё один паттерн, который всплыл в нескольких местах: вложенные *Select* с конструкторами внутри — это почти всегда нарушение SRP на уровне строки. Конструирование объекта и преобразование коллекции — разные ответственности. Когда они в одной строке, ни та ни другая не читается нормально.
