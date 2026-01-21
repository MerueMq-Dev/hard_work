# Отчёт: TDD и три уровня рассуждений

## Часть 1. Как я обычно писал код (наивный подход)

### Команда для редактирования настроек антифрода

```csharp
public class EditAntifraudSettingsRequest : IRequest
{
    public bool IsActive { get; set; }
    public RuleData[] Rules { get; set; }
}

// Обработчик команды
public class RequestHandler : IRequestHandler<EditAntifraudSettingsRequest>
{
    public async Task Handle(EditAntifraudSettingsRequest request)
    {
        var corporationId = await accountService.GetCorporation();
        var settings = await repository.Get(corporationId);

        if (settings == null)
        {
            settings = new AntifraudSettings(corporationId, request.IsActive, request.Rules);
        }
        else
        {
            settings.Update(request.IsActive, request.Rules);
        }

        await repository.Save(settings);
    }
}
```

### Тест который я написал

```csharp
[Fact]
public async Task Handle_WhenSettingsNull_CreatesNewSettings()
{
    var request = new EditAntifraudSettingsRequest 
    { 
        IsActive = true,
        Rules = new[] { new RuleData { Name = "Rule1", IsActive = true } }
    };
    
    _mockRepository.Setup(r => r.Get(It.IsAny<long>())).ReturnsAsync((AntifraudSettings?)null);
    _mockAccountService.Setup(s => s.GetCorporation()).ReturnsAsync(100L);
    
    var handler = new RequestHandler(_mockRepository.Object, _mockAccountService.Object);
    
    await handler.Handle(request);
    
    _mockRepository.Verify(r => r.Save(It.IsAny<AntifraudSettings>()), Times.Once);
}
```

---

## Часть 2. Как надо было делать (с пониманием трёх уровней)

### Сначала я сел и написал что вообще должно происходить

```
КОМАНДА: EditAntifraudSettings
ЧТО ДЕЛАЕТ: Меняет настройки системы антифрода для корпорации

ЧТО ДОЛЖНО ПОЛУЧИТЬСЯ ПОСЛЕ ВЫПОЛНЕНИЯ:
1. Флаг IsActive системы становится таким, какой передали в команде
2. Если в команде есть правило с существующим Id - обновляем это правило
3. Если в команде есть правило без Id (null) - создаём новое правило
4. Если какое-то правило было, но в команде его нет - помечаем как удалённое
5. UpdateDate обновляется на текущее время, ConcurrencyToken генерируется новый
```

### Доменная модель

```csharp
public class AntifraudSettings
{
    public long CorporationId { get; private set; }
    public bool IsActive { get; private set; }
    public List<AntifraudRule> Rules { get; set; } = new();
    public DateTimeOffset UpdateDate { get; private set; }
    public Guid ConcurrencyToken { get; private set; }
    
    public AntifraudSettings(long corporationId, bool isActive, IEnumerable<RuleData> rules)
    {
        CorporationId = corporationId;
        IsActive = isActive;
        ApplyRules(rules);
        UpdateDate = DateTimeOffset.UtcNow;
        ConcurrencyToken = Guid.NewGuid();
    }

    public void Update(bool isActive, IEnumerable<RuleData> rules)
    {
        IsActive = isActive;
        ApplyRules(rules);
        UpdateDate = DateTimeOffset.UtcNow;
        ConcurrencyToken = Guid.NewGuid();
    }

    private void ApplyRules(IEnumerable<RuleData> rules)
    {
        foreach (var rule in Rules) 
        {
            rule.MarkAsDeleted();
        }

        if (rules == null) return;

        foreach (var data in rules)
        {
            if (data.Id.HasValue)
            {
                var rule = Rules.FirstOrDefault(x => x.Id == data.Id);
                if (rule != null)
                    rule.Update(data.Name, data.IsActive);
            }
            else
            {
                Rules.Add(new AntifraudRule(data.Name, data.IsActive));
            }
        }
    }
}

public class AntifraudRule
{
    public AntifraudRule(string name, bool isActive) 
    {
        Id = Guid.NewGuid();
        Name = name;
        IsActive = isActive;
        IsDeleted = false;
    }

    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public bool IsActive { get; private set; }
    public bool IsDeleted { get; private set; }

    public void Update(string name, bool isActive)
    {
        Name = name;
        IsActive = isActive;
        IsDeleted = false;
    }

    public void MarkAsDeleted()
    {
        IsDeleted = true;
    }
}

public class RuleData
{
    public Guid? Id { get; set; }
    public string Name { get; set; }
    public bool IsActive { get; set; }
}
```

### Тесты которые проверяют спецификацию

```csharp
public class EditAntifraudSettingsSpecificationTests
{
    // Проверяем пункт 1 - можно включать и выключать систему
    [Fact]
    public async Task Spec_SystemCanBeActivatedAndDeactivated()
    {
        var request = new EditAntifraudSettingsRequest 
        { 
            IsActive = true, 
            Rules = Array.Empty<RuleData>() 
        };
        
        _mockRepository.Setup(r => r.Get(100)).ReturnsAsync((AntifraudSettings?)null);
        _mockAccountService.Setup(s => s.GetCorporation()).ReturnsAsync(100L);
        
        var handler = new RequestHandler(_mockRepository.Object, _mockAccountService.Object);
        var result = await handler.Handle(request);
        
        Assert.True(result.IsActive);
    }

    // Проверяем пункт 2 - правило с Id обновляется
    [Fact]
    public async Task Spec_ExistingRuleCanBeUpdated()
    {
        var ruleId = Guid.NewGuid();
        var existingSettings = new AntifraudSettings(100, true, new[]
        {
            new RuleData { Id = ruleId, Name = "OldName", IsActive = true }
        });
        
        var request = new EditAntifraudSettingsRequest 
        { 
            IsActive = true,
            Rules = new[]
            {
                new RuleData { Id = ruleId, Name = "NewName", IsActive = false }
            }
        };
        
        _mockRepository.Setup(r => r.Get(100)).ReturnsAsync(existingSettings);
        _mockAccountService.Setup(s => s.GetCorporation()).ReturnsAsync(100L);
        
        var handler = new RequestHandler(_mockRepository.Object, _mockAccountService.Object);
        var result = await handler.Handle(request);
        
        var updatedRule = result.Rules.FirstOrDefault(r => r.Id == ruleId);
        Assert.NotNull(updatedRule);
        Assert.Equal("NewName", updatedRule.Name);
        Assert.False(updatedRule.IsActive);
    }

    // Проверяем пункт 3 - правило без Id создаётся
    [Fact]
    public async Task Spec_NewRuleCanBeCreated()
    {
        var existingSettings = new AntifraudSettings(100, true, Array.Empty<RuleData>());
        
        var request = new EditAntifraudSettingsRequest 
        { 
            IsActive = true,
            Rules = new[]
            {
                new RuleData { Id = null, Name = "FraudPattern", IsActive = true }
            }
        };
        
        _mockRepository.Setup(r => r.Get(100)).ReturnsAsync(existingSettings);
        _mockAccountService.Setup(s => s.GetCorporation()).ReturnsAsync(100L);
        
        var handler = new RequestHandler(_mockRepository.Object, _mockAccountService.Object);
        var result = await handler.Handle(request);
        
        Assert.Single(result.Rules);
        Assert.Equal("FraudPattern", result.Rules.First().Name);
    }

    // Проверяем пункт 4 - что не пришло в команде, то удаляется
    [Fact]
    public async Task Spec_RuleNotInCommandIsDeleted()
    {
        var ruleToKeep = Guid.NewGuid();
        var ruleToDelete = Guid.NewGuid();
        
        var existingSettings = new AntifraudSettings(100, true, new[]
        {
            new RuleData { Id = ruleToKeep, Name = "Rule1", IsActive = true },
            new RuleData { Id = ruleToDelete, Name = "Rule2", IsActive = true }
        });
        
        var request = new EditAntifraudSettingsRequest 
        { 
            IsActive = true,
            Rules = new[]
            {
                new RuleData { Id = ruleToKeep, Name = "Rule1", IsActive = true }
            }
        };
        
        _mockRepository.Setup(r => r.Get(100)).ReturnsAsync(existingSettings);
        _mockAccountService.Setup(s => s.GetCorporation()).ReturnsAsync(100L);
        
        var handler = new RequestHandler(_mockRepository.Object, _mockAccountService.Object);
        var result = await handler.Handle(request);
        
        Assert.Single(result.Rules);
        Assert.DoesNotContain(result.Rules, r => r.Id == ruleToDelete);
    }
}
```

---

## Рефлексия

Я долго не понимал, почему мои тесты разваливаются после каждого рефакторинга. Вот был у меня тест ShouldUpdateAntifraudRule — он проверял, что _repository.Update вызвался с нужными параметрами. Решил изменить структуру агрегата, перенёс логику — тест сломался. Хотя с точки зрения бизнеса ничего не поменялось, правило как обновлялось, так и обновляется.

В общем, дошло до меня не сразу. Я проверял не то. Проверял, как код работает внутри, а надо было — что он делает.

Попробовал иначе на команде EditAntifraudSettings. Сел, открыл блокнот и выписал: что эта команда вообще должна гарантировать? Систему можно активировать — ок. Правило создаётся, если его не было. Обновляется, если было. А если правило не пришло в команде — удаляется. Всё, это требования бизнеса.

Написал под каждый пункт отдельный тест. CanActivateSystem, CanCreateRule, CanUpdateRule. Никаких моков репозитория, никаких проверок вызовов методов. Только: активировал систему — она активна? Создал правило — оно есть?

Потом начал рефакторить. Переделал внутреннюю логику применения правил — сначала все помечаются как удалённые, потом восстанавливаются те что пришли. Думал, половина тестов упадёт. Запустил — зелёные. Все. Контракт же не поменялся, изменилась только реализация.

Кстати, доменка стала чище. UpdateDate и ConcurrencyToken теперь обновляются прямо в методе Update, не надо помнить про них каждый раз. Всё в одном месте.

Теперь если новый человек откроет тесты на EditAntifraudSettings, он минут за десять разберётся что команда умеет. Без единой строчки кода, просто по названиям тестов и ассертам.

Потом прочитал материал про три уровня рассуждений. Там как раз это и объяснялось. Есть три слоя. Внизу — конкретные значения во время выполнения, какой-нибудь Guid = "123e4567...". Выше — код, как я это реализую. И ещё выше — спецификация, что система обязана делать. Я всегда начинал снизу, от кода. А надо сверху, от требований.

С TDD у меня поэтому не складывалось раньше. Я думал, это про "напиши тест, потом код". А на самом деле это про другое: сначала пойми, что должно работать, потом напиши тесты на это понимание, и только потом реализуй. Тесты и код не связаны напрямую, они оба следуют из спецификации.

Буду стараться делать так: открываю задачу, создаю файл postconditions.txt, пишу постусловия. Пока не напишу хотя бы три пункта — в IDE не захожу. Да, это тормозит. Но помогает. Если не могу сформулировать постусловие для метода — значит я ещё думаю на уровне "как сделать", а не "что должно получиться".

Вообще стало понятно, почему старые проекты гниют. Все пишут код, никто не фиксирует требования. Проходит полгода, код меняется десять раз, а что он должен был делать изначально — уже никто не помнит. Поэтому и баги.

Хороший пример — ConcurrencyToken. В коде это обычный Guid. Я его сравниваю, обновляю. Но на уровне спецификации это гарантия: два одновременных обновления не затрут друг друга. Требование одно, а реализовать можно по-разному — через версионирование, через timestamp, через что угодно.

Главное что я понял: TDD — это не про тесты первыми. Это про вопрос "что система должна делать" первым. Остальное потом.
