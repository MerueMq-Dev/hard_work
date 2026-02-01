# Увидеть ясную структуру дизайна

## Введение

Задание: выбрать куски кода из проекта, сформулировать их логический дизайн (уровень 3), проверить соответствие кода дизайну, переписать код декларативно с соответствием 1:1.

---
# 1: Списание бонусов - цепочка if'ов со статусами

## Исходный код (как было)

```csharp
internal static async Task ExpectWithdrawBonus(Calculation calculation)
{
    if (calculation.WithdrawBonusStatus == WithdrawStatusEnum.yes || 
        calculation.WithdrawBonusStatus == WithdrawStatusEnum.no)
        return;

    if (calculation.Guest.HasTelegram == false)
    {
        calculation.SetWithdrawBonusNoActionStatus(WithdrawStatusEnum.noTelegram);
        return;
    }

    if (calculation.Guest.Bonus.Balance == 0)
    {
        calculation.SetWithdrawBonusNoActionStatus(WithdrawStatusEnum.noBonuses);
        return;
    }

    if (calculation.GlobalBonusSettings?.IsActive == false)
        return;

    decimal maxByPercent = calculation.TotalAmount * 
        ((decimal?)calculation.GlobalBonusSettings?.Main?.PercentOfOrderPayment ?? 0) / 100;
    decimal availableByLimit = calculation.TotalLimitDiscount - calculation.TotalDiscount;
    decimal maxBonus = Math.Min(maxByPercent, availableByLimit);

    calculation.SetWithdrawBonusRequestedStatus((int)Math.Floor(Math.Max(0, maxBonus)) / 100);
}

internal static async Task CheckAndWithdrawBonus(Calculation calculation)
{
    if (calculation.WithdrawBonusStatus == WithdrawStatusEnum.noaction 
        || calculation.WithdrawBonusStatus == WithdrawStatusEnum.no 
        || calculation.WithdrawBonusStatus == WithdrawStatusEnum.requested)
        return;
    
    if (calculation.WithdrawBonusStatus == WithdrawStatusEnum.yes)
    {
        var discount = DiscountFactory.CreateAmountDiscount(
            Guid.NewGuid(), 
            $"бонусы: {calculation.WithdrawBonusActualAmount}", 
            calculation.WithdrawBonusActualAmount);

        Calculator.CalculateDiscount(calculation, discount, DiscountTypeEnum.bonus);
    }
}

// Enum статусов
[Flags]
public enum WithdrawStatusEnum
{
    noTelegram = 1,   // нет регистрации в ТГ боте
    noBonuses = 2,    // нет бонусов
    requested = 4,    // запросим одобрение списание в ТГ боте
    yes = 8,          // Гость согласился
    no = 16,          // Гость отказался
    noaction = noTelegram | noBonuses,
}
```

## Что не так с этим кодом?

Смотрю на *ExpectWithdrawBonus* и вижу цепочку из 4 проверок через early return. Каждая проверка возвращается, если условие не выполнено. Логика читается как "проверяем препятствия одно за другим".

```csharp
if (уже определился) return;
if (нет телеграма) return;
if (нет бонусов) return;
if (настройки выключены) return;
// Только если всё ок - рассчитываем
```

Проблема в том что эта последовательность проверок - это паттерн "валидация перед действием", но он не выражен явно. Просто куча if'ов. Непонятно какая логика: это guard clauses? Это условия применимости? Это валидация?

В *CheckAndWithdrawBonus* проверка через *||* трёх статусов. Комментарии говорят "если нет действия ИЛИ отказ ИЛИ ещё не определился - выходим". Но из кода это читается как магическое заклинание *noaction || no || requested*.

Enum с флагами *[Flags]* и комбинированным значением *noaction = noTelegram | noBonuses* добавляет сложности. Почему *noaction* это комбинация? Зачем флаги если значения не комбинируются по смыслу (гость не может одновременно согласиться И отказаться)?

Самое плохое - логика "когда можно списывать бонусы" размазана. В *ExpectWithdrawBonus* проверки одни, в *CheckAndWithdrawBonus* другие. Нет единого места где написано "вот условия списания бонусов".

## Что должно быть (формулирую дизайн)

**СИСТЕМА: Списание бонусов в оплату заказа**

**СТАТУСЫ ПРОЦЕССА:**
- **noTelegram** - гость не зарегистрирован в боте (нельзя спросить)
- **noBonuses** - у гостя нет бонусов (нечего списывать)
- **requested** - отправили запрос гостю, ждём ответа
- **yes** - гость согласился списать бонусы
- **no** - гость отказался от списания

**АЛГОРИТМ:**

1. **Проверка возможности списания:**
   - Если уже определились (yes/no) - ничего не делаем
   - Если нет телеграма - статус noTelegram
   - Если нет бонусов - статус noBonuses
   - Если настройки выключены - выход
   - Иначе - рассчитываем сумму и статус requested

2. **Применение списания:**
   - Если статус yes - списываем бонусы как скидку
   - Иначе - ничего не делаем

**ИНВАРИАНТЫ:**
- Бонусы списываются только при явном согласии (yes)
- Максимум списания ограничен процентом от суммы И лимитом скидок
- Один раз определились (yes/no) - повторно не проверяем

Вот это уровень 3. Видна последовательность принятия решения.

## Переписанный код (дизайн 1:1 виден в коде)

Выделяю концепции "причина отказа" и "возможность списания":

```csharp
internal static async Task ExpectWithdrawBonus(Calculation calculation)
{
    // Уже определились - не проверяем повторно
    if (calculation.WithdrawBonusStatus.IsDecided())
        return;

    // Проверяем возможность списания
    var canWithdraw = CanWithdrawBonuses(calculation);
    
    if (!canWithdraw.IsPossible)
    {
        calculation.SetWithdrawBonusNoActionStatus(canWithdraw.Reason);
        return;
    }

    // Рассчитываем максимальную сумму для списания
    var maxAmount = CalculateMaxWithdrawAmount(calculation);
    
    calculation.SetWithdrawBonusRequestedStatus(maxAmount);
}

internal static async Task CheckAndWithdrawBonus(Calculation calculation)
{
    // Списываем только если гость явно согласился
    if (calculation.WithdrawBonusStatus != WithdrawStatusEnum.yes)
        return;

    var discount = DiscountFactory.CreateAmountDiscount(
        Guid.NewGuid(), 
        $"бонусы: {calculation.WithdrawBonusActualAmount}", 
        calculation.WithdrawBonusActualAmount);

    Calculator.CalculateDiscount(calculation, discount, DiscountTypeEnum.bonus);
}

// Проверка возможности списания бонусов
private static WithdrawPossibility CanWithdrawBonuses(Calculation calculation)
{
    if (!calculation.Guest.HasTelegram)
        return WithdrawPossibility.No(WithdrawStatusEnum.noTelegram);

    if (calculation.Guest.Bonus.Balance == 0)
        return WithdrawPossibility.No(WithdrawStatusEnum.noBonuses);

    if (calculation.GlobalBonusSettings?.IsActive == false)
        return WithdrawPossibility.No(WithdrawStatusEnum.noaction);

    return WithdrawPossibility.Yes();
}

// Расчёт максимальной суммы для списания
private static int CalculateMaxWithdrawAmount(Calculation calculation)
{
    decimal maxByPercent = calculation.TotalAmount * 
        ((decimal?)calculation.GlobalBonusSettings?.Main?.PercentOfOrderPayment ?? 0) / 100;
    
    decimal availableByLimit = calculation.TotalLimitDiscount - calculation.TotalDiscount;
    
    decimal maxBonus = Math.Min(maxByPercent, availableByLimit);
    
    return (int)Math.Floor(Math.Max(0, maxBonus)) / 100;
}

// Value object для результата проверки
private record WithdrawPossibility(bool IsPossible, WithdrawStatusEnum? Reason)
{
    public static WithdrawPossibility Yes() => new(true, null);
    public static WithdrawPossibility No(WithdrawStatusEnum reason) => new(false, reason);
}

// Extension для проверки "определились или нет"
public static class WithdrawStatusEnumExtensions
{
    public static bool IsDecided(this WithdrawStatusEnum status)
    {
        return status == WithdrawStatusEnum.yes || 
               status == WithdrawStatusEnum.no;
    }
}

// Упрощённый enum без флагов
public enum WithdrawStatusEnum
{
    noTelegram,   // нет регистрации в ТГ боте
    noBonuses,    // нет бонусов  
    noaction,     // нет действия (настройки выключены)
    requested,    // запросили одобрение
    yes,          // согласился
    no            // отказался
}
```

## Что изменилось?

Цепочка if'ов превратилась в метод *CanWithdrawBonuses* который возвращает *WithdrawPossibility*. Теперь явно видно что это "проверка возможности", а не просто куча условий.

Value object *WithdrawPossibility* инкапсулирует результат: можно/нельзя + причина. Вместо *if (нет телеграма) return* теперь *if (!HasTelegram) return No(noTelegram)*. Декларативно.

Расчёт максимальной суммы вынесен в отдельный метод *CalculateMaxWithdrawAmount*. На уровне дизайна это отдельный шаг, значит в коде отдельный метод.

Extension *IsDecided()* делает проверку читаемой. Было: *status == yes || status == no*. Стало: *status.IsDecided()*. Намерение явное.

В *CheckAndWithdrawBonus* убрал проверку трёх статусов через *||*. Теперь просто *!= yes*. Логика стала проще: списываем только если явное согласие, иначе ничего.

Убрал *[Flags]* из enum потому что статусы не комбинируются по смыслу. *noaction* стал обычным значением, а не комбинацией флагов.

Основной метод *ExpectWithdrawBonus* стал декларативным - три шага видны явно:
1. *IsDecided()* - проверка что уже определились
2. *CanWithdrawBonuses()* - проверка возможности
3. *CalculateMaxWithdrawAmount()* - расчёт суммы

Соответствие 1:1: в дизайне "проверка возможности списания", в коде метод *CanWithdrawBonuses*. В дизайне "расчёт максимальной суммы", в коде *CalculateMaxWithdrawAmount*.

## Краткий вывод

Цепочка if'ов с early return - это паттерн валидации, но он не выражен явно в коде. Выделил метод *CanWithdrawBonuses* с value object *WithdrawPossibility*, расчёт суммы в отдельный метод. Убрал флаги из enum. Потратил около 80 минут.


---

# 2: Выбор самых выгодных акций - два одинаковых цикла

## Исходный код (как было)

```csharp
private static List GetMostValuablePromotions(List promotions, Calculation calculation)
{
    if (promotions == null || promotions.Count == 0)
        return new List();

    // Разделение на комбинируемые и одиночные акции
    var combinedPromotions = promotions.Where(p => p.IsCombined).ToArray();
    var singlePromotions = promotions.Where(p => !p.IsCombined).ToArray();

    // Фильтруем комбинируемые акции которые срабатывают
    List isMatchCombinedPromotions = new List();
    foreach(var p in combinedPromotions)
    {
        bool promoNotMatched = false;
        foreach(var c in p.Conditions)
        {
            if (c.IsMatch(calculation) == false)
            {
                promoNotMatched = true;
                break;
            }
        }

        if (promoNotMatched == false)
        {
            isMatchCombinedPromotions.Add(p);
        }
    }

    // Фильтруем одиночные акции которые срабатывают
    List isMatchSinglePromotions = new List();
    foreach (var p in singlePromotions)
    {
        bool promoNotMatched = false;
        foreach (var c in p.Conditions)
        {
            if (c.IsMatch(calculation) == false)
            {
                promoNotMatched = true;
                break;
            }
        }

        if (promoNotMatched == false)
        {
            isMatchSinglePromotions.Add(p);
        }
    }

    // Суммарная ценность комбинируемых акций
    decimal combinedTotalWorth = isMatchCombinedPromotions
        .Sum(p => p.Reward?.Worth(calculation) ?? 0);

    // Поиск максимальной ценности среди одиночных акций
    decimal maxSingleWorth = isMatchSinglePromotions.Any()
        ? isMatchSinglePromotions.Max(p => p.Reward?.Worth(calculation) ?? 0)
        : decimal.MinValue;

    // Сравнение и выбор результата
    if (isMatchCombinedPromotions.Count > 0 && combinedTotalWorth > maxSingleWorth)
    {
        return isMatchCombinedPromotions;
    }
    else
    {
        var max = isMatchSinglePromotions.Any() 
            ? isMatchSinglePromotions.MaxBy(p => p.Reward.Worth(calculation)) 
            : null;

        if (max == null)
            return new List();

        return [max];
    }
}
```

## Что не так с этим кодом?

Вижу два абсолютно одинаковых цикла - один для combinedPromotions, второй для singlePromotions. Оба делают одно: проверяют подходит ли акция под расчёт.

```csharp
bool promoNotMatched = false;
foreach(var c in p.Conditions)
{
    if (c.IsMatch(calculation) == false)
    {
        promoNotMatched = true;
        break;
    }
}
if (promoNotMatched == false)
{
    isMatchCombinedPromotions.Add(p);  // Единственное отличие - куда добавляем
}
```

Это классическая копипаста. Если завтра изменится логика проверки условий - придётся менять в двух местах. Уже вижу потенциальный баг: исправили первый цикл, забыли про второй.

Хуже то что логика "проверка всех условий акции" размазана по коду. Это не метод, не функция - это просто повторяющийся кусок императивного кода. На уровне 3 это должна быть концепция "акция подходит для расчёта".

Название переменной promoNotMatched с двойным отрицанием запутывает. if (promoNotMatched == false) читается как "если акция НЕ НЕ подходит". Надо думать что это значит.

Дальше идёт сравнение по ценности - суммарная ценность комбинируемых vs максимальная одиночная. Эта бизнес-логика тоже не видна явно. Почему именно так? Из кода непонятно.

## Что должно быть (формулирую дизайн)

**СИСТЕМА: Выбор акций для применения**

**ДВА ТИПА АКЦИЙ:**
1. **Комбинируемые (IsCombined=true)** - можно применять несколько одновременно
2. **Одиночные (IsCombined=false)** - только одна самая выгодная

**АЛГОРИТМ ВЫБОРА:**
1. Отфильтровать акции которые подходят (все условия выполнены)
2. Для комбинируемых: посчитать суммарную ценность
3. Для одиночных: найти максимальную ценность
4. Выбрать что выгоднее: все комбинируемые или одна самая выгодная одиночная

**ИНВАРИАНТЫ:**
- Акция подходит только если ВСЕ её условия выполнены
- Комбинируемые акции суммируются
- Одиночные акции взаимоисключающие (только одна)
- Выбираем вариант с максимальной выгодой для клиента

Вот это уровень 3. Видно ЧТО делается и ПОЧЕМУ так.

## Переписанный код (дизайн 1:1 виден в коде)

Выделяю концепцию "проверка условий акции":

```csharp
private static List GetMostValuablePromotions(List promotions, Calculation calculation)
{
    if (promotions == null || promotions.Count == 0)
        return new List();

    // Фильтруем акции которые подходят под расчёт
    var matchingPromotions = promotions
        .Where(p => AllConditionsMatch(p, calculation))
        .ToList();

    if (!matchingPromotions.Any())
        return new List();

    // Разделяем на комбинируемые и одиночные
    var combinedPromotions = matchingPromotions.Where(p => p.IsCombined).ToList();
    var singlePromotions = matchingPromotions.Where(p => !p.IsCombined).ToList();

    // Выбираем наиболее выгодный вариант
    return ChooseMostValuable(combinedPromotions, singlePromotions, calculation);
}

// Проверка что ВСЕ условия акции выполнены
private static bool AllConditionsMatch(Promotion promotion, Calculation calculation)
{
    return promotion.Conditions.All(c => c.IsMatch(calculation));
}

// Выбор между комбинируемыми и одиночными акциями
private static List ChooseMostValuable(
    List combined, 
    List single, 
    Calculation calculation)
{
    var combinedWorth = CalculateTotalWorth(combined, calculation);
    var singleWorth = FindMaxWorth(single, calculation);

    // Комбинируемые выгоднее
    if (combined.Any() && combinedWorth > singleWorth)
        return combined;

    // Одиночная выгоднее
    var bestSingle = single.MaxBy(p => p.Reward?.Worth(calculation) ?? 0);
    return bestSingle != null ? [bestSingle] : new List();
}

// Суммарная ценность комбинируемых акций

private static decimal CalculateTotalWorth(List promotions, Calculation calculation)
{
    return promotions.Sum(p => p.Reward?.Worth(calculation) ?? 0);
}

// Максимальная ценность среди одиночных акций
private static decimal FindMaxWorth(List promotions, Calculation calculation)
{
    return promotions.Any()
        ? promotions.Max(p => p.Reward?.Worth(calculation) ?? 0)
        : decimal.MinValue;
}
```

## Что изменилось?

Два одинаковых цикла превратились в один вызов AllConditionsMatch. Теперь логика "все условия должны выполниться" это отдельная функция, а не повторяющийся код.

Использовал LINQ .All() вместо императивного цикла с флагом promoNotMatched. Читается прямо: "все условия подходят". Никаких двойных отрицаний.

Сначала фильтрую все акции, потом разделяю на комбинируемые/одиночные. В старом коде было наоборот - сначала разделение, потом фильтрация двумя циклами. Теперь один фильтр для всех.

Выделил метод ChooseMostValuable который инкапсулирует бизнес-логику выбора. Теперь видно что это отдельный шаг алгоритма: "выбрать наиболее выгодный вариант".

Методы CalculateTotalWorth и FindMaxWorth делают дизайн явным. Для комбинируемых - сумма, для одиночных - максимум. Это видно из названий методов, не нужно читать реализацию.

Основной метод стал декларативным - три шага видны явно:
1. AllConditionsMatch - отфильтровать подходящие
2. Where(p => p.IsCombined) - разделить на типы
3. ChooseMostValuable - выбрать выгоднейший вариант

Соответствие 1:1: в дизайне три шага алгоритма, в коде три метода. В дизайне "все условия должны выполниться", в коде .All(c => c.IsMatch(calculation)).

## Краткий вывод

Два одинаковых цикла - это не "фильтрация комбинируемых и одиночных", это одна операция "проверка условий" применённая дважды. Выделил в метод AllConditionsMatch, использовал LINQ вместо императивных циклов. Потратил где-то 40-45 минут. 

---


## Рефлексия
Раньше я не понимал, почему код выглядит рабочим, но ощущение беспорядка всё равно остаётся. Любое изменение превращалось в лотерею: тронешь одно место — что-то ломается в другом. Особенно это ощущалось там, где логика вырастала из цепочек условий и циклов. Всё по делу, всё выполняется, но чтобы внести изменение, каждый раз приходится заново разбираться, что именно здесь происходит.

Осознание пришло не сразу. Я писал код на уровне «как это сделать», а не «что это вообще такое».

В задаче с акциями я видел два цикла — для комбинируемых и одиночных акций — и воспринимал их как две разные логики. На самом деле логика была одна: акция подходит, если выполнены все её условия. Я оперировал реализацией (два фрагмента кода), а не моделью (понятие «подходящая акция»).

Я попробовал действовать иначе. Вместо немедленного рефакторинга я сначала сформулировал словами, что происходит на уровне смысла. Сначала определяется, какие акции вообще подходят под расчёт. Затем из них выбирается наиболее выгодный вариант. Проверка условий — это самостоятельный шаг алгоритма, а не деталь конкретного цикла.

Когда это было зафиксировано в виде текста, стало очевидно, что два цикла — это ложная структура. В коде должна существовать явная операция проверки условий. Так появился метод AllConditionsMatch, и структура кода начала отражать структуру алгоритма, а не случайное разбиение коллекций.

Во второй задаче проблема проявлялась иначе. Там не было копипасты, но была длинная цепочка if-ов при списании бонусов. Я воспринимал её как «набор проверок». Однако на уровне смысла это было единое правило: можно ли вообще списывать бонусы и, если нельзя, по какой причине.

В системе существовали понятия «возможность списания» и «причина отказа», но в коде они отсутствовали — вместо них была последовательность условий с return. Когда появилась явная модель результата проверки (можно / нельзя + причина), код перестал быть набором барьеров и стал описанием правила системы.

Раньше я бы ограничился тем, что «вынес логику в методы». Но ключевое изменение произошло раньше: сначала появилась формулировка на уровне модели — «подходящая акция», «возможность списания бонусов» — и только потом код был приведён в соответствие с этими понятиями. Сначала модель, затем реализация.

После этого я иначе стал понимать декларативность. Ранее она ассоциировалась у меня с использованием LINQ или «функционального стиля». На практике же это вопрос уровня абстракции. Декларативный код называет сущности модели и правила системы. Императивный — оперирует циклами, флагами и условиями, не поднимаясь до уровня смысла.

Стало заметно, что самые полезные 30–50 минут работы проходят вне IDE. Когда я формулирую словами шаги алгоритма или правила системы, структура кода затем выстраивается почти автоматически. Если я не могу описать словами, что делает фрагмент логики, значит, я всё ещё думаю на уровне «как выполняется», а не «что означает».

Теперь стало понятнее, почему код со временем деградирует. В нём остаётся только уровень реализации. Уровень модели — смысл операций, инварианты, правила — существует лишь в головах разработчиков. Люди уходят, контекст теряется, код остаётся. В результате он превращается в набор решений, которые трудно менять.

Главный вывод для меня — хорошая структура кода связана не с «красотой», а с совпадением структуры программы со структурой предметной области. Если в системе есть правило, оно должно быть выражено как правило в коде. Если есть понятие, у него должно быть явное имя. Не наоборот.

Теперь главный вопрос перед любым фрагментом логики для меня звучит так:
«Какая сущность или правило системы здесь реализуется, а не какой именно код здесь написан?»

Если я не могу ответить на этот вопрос, значит, к написанию метода я ещё не до конца готов.