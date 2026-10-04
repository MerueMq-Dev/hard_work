## Чужие умолчания и типы, которые не дают на них полагаться

Третий пункт материала про ускорение фреймворков — про порядок. ORM отдаёт строки отсортированными по id, код дальше на это опирается, и через какое-то время уже никто не помнит, было это обещано или просто так совпало. Совет там простой: в реализации быть конкретным, а в спецификации не обещать лишнего. Если порядок не важен, надо возвращать множество, а не список, — тогда на порядок никто и не сможет опереться.

Сначала расскажу про случаи в MusicSync, где я опирался на чужое поведение по умолчанию, а оно оказалось другим. Потом — про места, где я поменял тип результата так, чтобы его нельзя было обработать неправильно.

### Часть 1. Где я полагался на чужие умолчания

#### 1. Плейлист — это не множество

Этот случай прямо про пункт 3, только вместо порядка тут повторы.

`GetPlaylistTracksAsync` у Spotify возвращает `IReadOnlyList<TrackDto>`. Это список, так что в нём допустимы и порядок, и повторы. Тип честный. А вот домен дальше читает его как множество:

```csharp
if (_matches.Exists(m => m.SourceTrackId == sourceTrackId))
    throw new DomainException($"Трек {sourceTrackId} уже обработан.");
```

И это не одна случайная строчка. Кроме проверки в `TransferJob.RecordMatch`, то же самое записано в уникальном индексе `(transfer_job_id, source_track_id)` в первой миграции и в тесте `RecordMatch_SameTrackTwiceThrows`.

По моей спецификации один и тот же трек на двух позициях — ошибка. Для Spotify это нормальный плейлист: трек туда можно добавить несколько раз, и API вернёт каждое вхождение отдельным элементом. На втором вхождении `RecordMatch` бросит исключение, `ProcessTransferJob.RunAsync` его поймает и переведёт весь перенос в `Failed` с текстом «Трек … уже обработан». Один повтор в плейлисте на двести треков — и не переносится ничего. В Spotify при этом остаётся пустой плейлист, потому что он создаётся ещё до матчинга.

Ошибка здесь моя: Spotify нигде не обещал, что треки в плейлисте уникальны. Это я сам придумал, когда писал инвариант, и просто не подумал, что так бывает.

Элемент плейлиста определяется позицией, поэтому и уникальность надо проверять по ней:

```csharp
if (_matches.Exists(m => m.Position == position))
    throw new DomainException($"Позиция {position} уже обработана.");
```

Индекс меняется так же. В `TrackMatchConfiguration` вместо

```csharp
builder.HasIndex(x => new { x.TransferJobId, x.SourceTrackId }).IsUnique();
```

должно быть

```csharp
builder.HasIndex(x => new { x.TransferJobId, x.Position }).IsUnique();
```

и под это новая миграция. Тогда повторы из исходного плейлиста попадут и в целевой: `GetAcceptedTrackIdsAsync` вернёт оба вхождения, и копия будет совпадать с оригиналом.

#### 2. Токен-эндпоинт Spotify не принимает JSON

Вот как обновляется токен в `SpotifyTokenRefreshService`:

```csharp
var response = await http.PostAsJsonAsync(TokenUrl, new
{
    grant_type = "refresh_token",
    refresh_token = account.RefreshToken
}, cancellationToken);
```

Я сначала ничего странного тут не видел: в `SpotifyMusicService` плейлисты создаются и наполняются тем же `PostAsJsonAsync`, и там всё работает. Но это `api.spotify.com`. Токены выдаёт `accounts.spotify.com/api/token`, который работает по OAuth 2.0, и параметры запроса туда передаются в формате `application/x-www-form-urlencoded`. В руководстве Spotify по code flow показано то же самое, а ссылка на это руководство стоит прямо в XML-комментарии этого класса. Та же ошибка есть в `SpotifyOAuthService.ExchangeCodeAsync`, где код меняется на токены.

Из-за этого обмен кода не проходит, и подключить Spotify через OAuth нельзя вообще. Обновление токена тоже не работает: `MusicServiceProvider` глотает ошибку, пока старый токен жив, а когда он протухает, просит подключить аккаунт заново. Для пользователя это выглядит так, будто Spotify иногда разлогинивает.

Правильно так:

```csharp
var response = await http.PostAsync(TokenUrl, new FormUrlEncodedContent(new Dictionary<string, string>
{
    ["grant_type"] = "refresh_token",
    ["refresh_token"] = account.RefreshToken
}), cancellationToken);
```

И в `ExchangeCodeAsync`:

```csharp
var response = await http.PostAsync(TokenUrl, new FormUrlEncodedContent(new Dictionary<string, string>
{
    ["grant_type"] = "authorization_code",
    ["code"] = code,
    ["redirect_uri"] = redirectUri
}), cancellationToken);
```

Тут тоже ошибся я: формат описан и в RFC, и у Spotify, а документация была в одном клике от кода.

#### 3. `timestamptz` приходит как `DateTime`

Этот случай уже был в прошлом разборе, но там я написал только о том, как починил. Здесь интереснее, кто виноват.

В базе колонки `timestamp with time zone`, в коде везде `DateTimeOffset`, и я считал, что они совпадают. Стенд упал на первом же `GetByIdAsync`:

```
A parameterless default constructor or one matching signature ... is required
```

Причин две. Npgsql начиная с шестой версии читает `timestamptz` как `DateTime` в UTC — это описано в их документации про дату и время, так что тут ошибся я. А Dapper, собирая запись через конструктор, требует, чтобы типы параметров совпадали с типами колонок в ридере. Такого правила я в документации не нашёл: readme в пакете Dapper 2.1.79 — это двадцать восемь строк, и про конструкторы и обработчики типов там ничего нет. Само правило я понял из текста исключения. Так что вина в основном моя, но отчасти и документации Dapper.

Починил обработчиком типа `DateTimeOffsetTypeHandler`, и здесь пункт 3 всплывает ещё раз. То, что зарегистрированный обработчик участвует в подборе конструктора, я знаю только потому, что это сработало. Никто этого не обещал, и если в следующей версии Dapper перестанет так делать, всё упадёт точно так же. Сейчас это проверяет только стенд, который поднимает агрегат на живой базе.

### Часть 2. Типы результата, которые исключают неправильную обработку

#### 1. Результат есть только у решения, которое применилось

Раньше репозиторий возвращал запись с кодом исхода, и хендлер должен был не забыть её проверить:

```csharp
var resolution = await transferJobs.ApplyDecisionAsync(jobId, userId, decision, cancellationToken);

resolution.EnsureApplied();

if (resolution.ReadyToFinalize)
    jobQueue.EnqueueTransferFinalization(jobId);
```

Если убрать строку с `EnsureApplied()`, код всё равно соберётся, и API ответит `204 No Content` на решение, которое не применилось. Такой же симптом был у бага с пропавшим UPDATE из прошлого разбора: API отвечал «успех», а в базе ничего не менялось. Наступать на это второй раз не хотелось.

Теперь репозиторий возвращает значение только при успехе:

```csharp
public sealed record AppliedMatchDecision
{
    internal AppliedMatchDecision(Guid matchId, bool readyToFinalize) { ... }

    public Guid MatchId { get; }
    public bool ReadyToFinalize { get; }
}
```

Конструктор `internal`, поэтому собрать такой объект могут только домен, инфраструктура и тесты, из слоя приложения этого не сделать. При отказе летит `MatchDecisionRejectedException` с причиной из перечисления `MatchDecisionRejection`, в котором варианта «применено» нет. Хендлер стал короче:

```csharp
var applied = await transferJobs.ApplyDecisionAsync(jobId, userId, decision, cancellationToken);

if (applied.ReadyToFinalize)
    jobQueue.EnqueueTransferFinalization(jobId);
```

Отдельной проверки в нём больше нет — если метод вернул значение, решение применилось. Для этого пришлось сделать `DomainException` незапечатанным, чтобы от него могло наследоваться новое исключение. API по-прежнему отвечает 400 с теми же текстами, это закреплено тестом на каждую причину.

C# не заставляет смотреть на возвращённое значение, так что результат всё ещё можно проигнорировать. Но при неудаче вылетит исключение, и `UnitOfWorkBehavior` откатит транзакцию.

#### 2. Аккаунты пользователя по сервисам

В прошлом разборе у меня остался открытый вопрос: зачем в `GetAllAsync` стоит `order by connected_at desc`. Ответа я так и не нашёл, а материал для таких случаев советует вообще не обещать порядок.

Было:

```csharp
Task<IReadOnlyList<ConnectedAccount>> GetAllAsync(Guid userId, CancellationToken cancellationToken = default);
```

```sql
where user_id = @UserId and is_active order by connected_at desc
```

Стало:

```csharp
Task<IReadOnlyDictionary<ServiceType, ConnectedAccount>> GetAllAsync(
    Guid userId,
    CancellationToken cancellationToken = default);
```

```sql
where user_id = @UserId and is_active
```

В словаре по сервису порядка нет, и взять `accounts[0]` как «самый свежий» уже не выйдет. Ключ — сервис, а завести больше одного активного аккаунта на сервис не даёт уникальный индекс `(user_id, service) where is_active`. Если индекс всё-таки как-то обошли и аккаунтов два, репозиторий падает, а не выбирает молча один из них:

```csharp
if (!accounts.TryAdd(account.Service, account))
    throw new InvalidOperationException(
        $"У пользователя {userId} больше одного активного аккаунта {account.Service}.");
```

Вызывающих у метода пока нет, так что контракт я поменял заранее, пока от порядка ещё ничего не зависит. Экрану «мои подключённые сервисы» этот метод наверняка понадобится.

#### 3. Проверка подключения без агрегата

`StartTransferHandler` перед запуском проверяет, что оба аккаунта подключены. Раньше это выглядело так:

```csharp
var account = await accounts.GetAsync(userId, service, cancellationToken);

if (account is null || !account.IsActive)
    throw new DomainException($"Аккаунт {service} не подключён.");

if (account.IsExpired && account.RefreshToken is null)
    throw new DomainException($"Токен доступа к {service} истёк, подключите аккаунт заново.");
```

`GetAsync` возвращает агрегат и регистрирует для него persist-делегат, поэтому проверка заодно ещё и пишет: при коммите команды на каждый аккаунт уходит UPDATE со всеми токенами, хотя ничего не поменялось. А у агрегата ещё есть `RefreshWith` и `Disconnect`: если вызвать их на объекте, поднятом «только посмотреть», изменение молча сохранится. `!account.IsActive` здесь ничего не делает, потому что `GetAsync` и так отбирает только активные аккаунты.

Теперь для этого есть отдельный метод и отдельный тип:

```csharp
public sealed record AccountConnectionStatus(ServiceType Service, DateTimeOffset? ExpiresAt, bool HasRefreshToken)
{
    public bool IsExpired => ConnectedAccount.IsExpiredAt(ExpiresAt, DateTimeOffset.UtcNow);
}
```

```csharp
var connection = await accounts.FindConnectionAsync(userId, service, cancellationToken)
                 ?? throw new DomainException($"Аккаунт {service} не подключён.");

if (connection.IsExpired && !connection.HasRefreshToken)
    throw new DomainException($"Токен доступа к {service} истёк, подключите аккаунт заново.");
```

У записи нет методов, которые что-то меняют, и репозиторий не регистрирует для неё persist, так что проверка ничего в базу не пишет. Правило «токен истёк» я вынес в `ConnectedAccount.IsExpiredAt`, чтобы `AccountConnectionStatus` и агрегат считали его одинаково; это сверяет тест.

Стенд проверяет это через `xmin` — служебную колонку Postgres, которая меняется при любом UPDATE строки, даже если значения те же. После `FindConnectionAsync` и коммита `xmin` у строки аккаунта прежний, после `GetAsync` и коммита — новый.

`MusicServiceProvider` по-прежнему работает с агрегатом: он обновляет токен и должен записать новый.
