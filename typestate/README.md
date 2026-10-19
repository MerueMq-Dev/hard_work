## Состояния как типы

Материал про Typestate-Oriented Programming предлагает с этим бороться так: каждое состояние объекта — отдельный тип со своими методами, а переход возвращает объект следующего состояния. Тогда вызов недопустимой операции просто не скомпилируется. В C# такой поддержки нет, но похожее можно собрать из обычных классов: на каждое состояние свой класс, а переход — метод, который возвращает объект другого класса.

Ниже три места, где я так сделал, и что из этого реально получилось.

### 1. Жизненный цикл переноса

*TransferJob* проходит через *Created → Running → (AwaitingReview) → Completed | Failed | Cancelled*. Раньше все его методы были публичными, и каждый сам проверял, можно ли его сейчас вызвать:

```csharp
public void Complete()
{
    EnsureNotTerminal();

    if (Status is not (TransferStatus.Running or TransferStatus.AwaitingReview))
        throw new DomainException("Завершать можно только запущенный перенос.");

    if (!AllTracksProcessed)
        throw new DomainException($"Обработано {ProcessedTracks} треков из {TotalTracks}, перенос ещё не готов.");

    if (HasPendingReview)
        throw new DomainException("Есть совпадения, ожидающие решения пользователя.");
    ...
}
```

То есть *job.Complete()* сразу после *Start()* или *RecordMatch* после *Cancel()* спокойно компилируются, а ошибка вылезет только в рантайме. В *ProcessTransferJob* за порядком следили ручные проверки статуса:

```csharp
if (job.Status != TransferStatus.Created)
    return;
...
job.Start(tracks.Count, destinationPlaylistId);
await MatchAllAsync(job, tracks, destination, cancellationToken);

if (job.HasPendingReview)
    return;
...
await AddAcceptedTracksAsync(job.Id, job.DestinationPlaylistId!, destination, cancellationToken);
job.Complete();
```

Я добавил переносу свойство *State*:

```csharp
public TransferState State => Status switch
{
    TransferStatus.Created => new CreatedTransfer(this),
    TransferStatus.Running when !AllTracksProcessed => new MatchingTransfer(this),
    TransferStatus.Running when HasPendingReview => new AwaitingReviewTransfer(this),
    TransferStatus.Running => new ReadyTransfer(this),
    TransferStatus.AwaitingReview => new AwaitingReviewTransfer(this),
    _ => new FinishedTransfer(this)
};
```

Тут три разные вещи, и их легко перепутать. *Status* — это колонка в базе, её пишет репозиторий. *State* — это свойство, которое при каждом обращении заново вычисляется из *Status* и загруженных совпадений и возвращает новый объект. А классы *CreatedTransfer*, *MatchingTransfer* и остальные определяют, какие операции вообще можно вызвать.

У *CreatedTransfer* есть только *Start*, и он возвращает *MatchingTransfer*. У *MatchingTransfer* есть *RecordMatch*. Решать спорные совпадения можно только в *AwaitingReviewTransfer*, завершать — только в *ReadyTransfer*, а в *FinishedTransfer* методов нет совсем. Методы самого агрегата стали *internal*, поэтому за пределами доменной сборки перенос меняется только через эти классы.

Статус «запущен» в базе один, но состояний из него вышло два: треки ещё обрабатываются, или всё уже готово к завершению. Миграция для этого не понадобилась.

Порядок веток в *switch* важен, и его стоит проверить отдельно. Пока не все треки обработаны, спорные совпадения уже могут быть: *!AllTracksProcessed* и *HasPendingReview* вполне истинны одновременно. Первая ветка выигрывает, и такой перенос считается *MatchingTransfer*. Это не прячет ожидание решения пользователя, а повторяет правило самого агрегата: *RefreshReviewState* переводит перенос в *AwaitingReview* только после того, как обработаны все треки. Пока матчинг идёт, ждать пользователя рано.

Вторая ветка, *Running* с обработанными треками и спорными совпадениями, при согласованных данных не встречается. Агрегат в такой ситуации сразу ставит *AwaitingReview*. Ветка страхует от рассогласования *Status* и счётчиков, и в этом случае *State* показывает не то, что лежит в колонке. Обратного запаса нет: *AwaitingReview* без спорных совпадений всё равно станет *AwaitingReviewTransfer*, и завершить такой перенос через состояния будет нельзя. Последняя ветка *_* относит к *FinishedTransfer* не только три терминальных статуса, но и любое неизвестное значение enum. [Уточнить: не лучше ли на неизвестный статус бросать исключение.]

У этого решения есть побочный эффект, его стоит назвать прямо. Раньше *ResolveManually* и *Skip* у агрегата были доступны в любом незавершённом состоянии. Теперь через состояния они есть только у *AwaitingReviewTransfer*. Рабочий путь решений идёт через SQL в *ApplyDecisionAsync*, и его это не коснулось, но API агрегата стал строже, чем был.

*Fail* и *Cancel* лежат в общем базовом классе *ActiveTransfer*. Я проверил по агрегату: оба метода разрешены в любом незавершённом состоянии, ограничение у них одно — *EnsureNotTerminal*. Так что сейчас наследование честное. Но это решение на будущее: любое новое незавершённое состояние автоматически получит отмену. Если появится шаг, который нельзя прерывать, например досыл треков в целевой плейлист, *ActiveTransfer* придётся разделить на более узкие интерфейсы вроде «можно отменить» и «можно провалить».

Фоновая обработка после переделки:

```csharp
if (job.State is not CreatedTransfer created)
    return;
...
var matching = created.Start(tracks.Count, destinationPlaylistId);
await MatchAllAsync(matching, tracks, destination, cancellationToken);

if (job.State is not ReadyTransfer ready)
    return;
...
await AddAcceptedTracksAsync(job.Id, ready.DestinationPlaylistId, destination, cancellationToken);
ready.Complete();
```

Ушёл *DestinationPlaylistId!*. В *ReadyTransfer* перенос всегда уже запущен и целевой плейлист создан, поэтому *!* теперь стоит внутри этого класса. *CancelTransferHandler* тоже работает через состояние: если перенос не *ActiveTransfer*, отменять там уже нечего.

Приятно было, что после того как я сделал методы агрегата *internal*, компилятор сам перечислил все места, где перенос меняли напрямую. Их было пять: фоновая обработка, отмена, два файла интеграционных тестов и стенд.

Теперь о том, что эти типы гарантируют, а что нет. Компилятор гарантирует две вещи: снаружи доменной сборки нельзя вызвать метод агрегата в обход состояний, а с объектом конкретного состояния доступны только его операции. Внутри доменной сборки *internal* ничего не запрещает, тесты с *InternalsVisibleTo* тоже ходят напрямую.

Не гарантирует компилятор свежесть объекта состояния. Все классы — обёртки над одним изменяемым агрегатом. После перехода старая обёртка никуда не девается и продолжает смотреть на тот же агрегат. В *MatchAllAsync* после последнего *RecordMatch* объект *matching* уже устарел: перенос перешёл в *ReadyTransfer* или *AwaitingReviewTransfer*, а метод *RecordMatch* у старого объекта остался. В материале на этот случай есть аннотация *unique*, которая запрещает держать вторую ссылку на объект. В C# нет ни её, ни линейных типов, которые позволили бы «израсходовать» объект при переходе. Поэтому проверки внутри *TransferJob* я не убирал, они остаются второй линией.

Ещё дальше от типов конкурентный доступ. Объект состояния вычислен из того, что было в памяти на момент загрузки. Если перенос тем временем отменили через API, фоновая обработка об этом не узнает. Этот случай ловит только охраняемый UPDATE в репозитории, и интеграционный тест *StaleWorker_DoesNotOverwriteCancellation* проверяет его на живой базе.

### 2. OAuth-флоу Spotify

Подключение Spotify идёт по шагам: подписать state, отправить пользователя на авторизацию, проверить state в callback, обменять код на токены, запросить профиль и сохранить аккаунт. Раньше все шаги принимали обычные строки:

```csharp
string GetAuthorizationUrl(string state);
Task<SpotifyTokenResult> ExchangeCodeAsync(string code, CancellationToken cancellationToken = default);
Task<SpotifyUserProfile> GetUserProfileAsync(string accessToken, CancellationToken cancellationToken = default);
```

Можно было вставить в ссылку неподписанную строку, обменять код без проверки state или собрать команду с *userId* откуда угодно. Правильный порядок был только в коде эндпоинта.

Сейчас у каждого шага свой тип результата, и следующий шаг принимает именно его:

```csharp
OAuthState GenerateState(Guid userId);
VerifiedOAuthState? ValidateState(string? state);

string GetAuthorizationUrl(OAuthState state);
Task<SpotifyTokens> ExchangeCodeAsync(VerifiedOAuthState state, string code, CancellationToken cancellationToken = default);
Task<SpotifyAuthorization> GetUserProfileAsync(SpotifyTokens tokens, CancellationToken cancellationToken = default);
```

У всех этих классов конструкторы *internal* и свойства только для чтения, поэтому в эндпоинте подделать их не получится. Пользователь дальше идёт вместе с токенами, и команда собирается на последнем шаге:

```csharp
var verified = stateService.ValidateState(state);
if (verified is null)
    return Results.BadRequest(...);

var tokens = await oauthService.ExchangeCodeAsync(verified, code, cancellationToken);
var authorization = await oauthService.GetUserProfileAsync(tokens, cancellationToken);

var accountId = await sender.Send(authorization.ToCommand(), cancellationToken);
```

Если переставить эти шаги местами, код не соберётся. Это ровно то, к чему призывает материал: порядок выполнения задаётся синтаксисом.

Важно точно понимать, что каждый тип обещает.

*OAuthState* — строка, которую подписал *OAuthStateService*. Код за пределами Infrastructure не может положить в ссылку произвольную строку.

*VerifiedOAuthState* — state из callback, у которого в момент проверки сошлась подпись и не вышел пятиминутный срок. Однократность использования он не гарантирует. Схема stateless, учёта использованных state нет, поэтому один и тот же state проходит проверку повторно, пока не истечёт срок. Повторный обмен по сути запрещает Spotify: по OAuth 2.0 код авторизации одноразовый, но это гарантия сервиса, а не наших типов. Код из callback к state тоже не привязан: это отдельная строка, которую мы просто передаём рядом. Для самого запроса на обмен *VerifiedOAuthState* не нужен — я передаю его как подтверждение, что проверку прошли. [Уточнить: нужен ли учёт использованных state, если повтор в течение пяти минут считается риском.]

*SpotifyTokens* — токены, которые вернул успешный обмен, плюс пользователь из проверенного state. Что токены рабочие и не будут отозваны, тип не знает. Что обмен вообще проходит, зависит от реализации запроса, а не от типов. [Уточнить: запрос на обмен сейчас отправляется в JSON, а токен-эндпоинт по OAuth 2.0 ждёт *application/x-www-form-urlencoded*; на живом Spotify это не проверялось.]

*SpotifyAuthorization* — профиль, запрошенный этими токенами. *ToCommand()* берёт пользователя из state, а не со стороны.

У этих типов есть два общих ограничения. *ValidateState* возвращает *VerifiedOAuthState?*, и передать результат без проверки на *null* компилятор не даст без предупреждения. Но в проекте *TreatWarningsAsErrors* выключен, так что это именно предупреждение, а не ошибка. И объекты одноразовыми не становятся: тот же *VerifiedOAuthState* в коде можно передать в *ExchangeCodeAsync* дважды — это та же проблема с отсутствием *unique*, что и в первом примере.

### 3. Клиент сервиса только из живого токена

Раньше фабрика клиентов принимала сам аккаунт:

```csharp
IMusicService Create(ConnectedAccount account);
```

Перед этим *MusicServiceProvider* должен проверить токен и, если надо, обновить его. Но это держалось только на ветвлениях в провайдере, а фабрика приняла бы любой аккаунт, хоть отключённый. И тут я заметил неприятную вещь: *Disconnect()* обнуляет срок токена, поэтому у отключённого аккаунта *IsExpired* равен *false*, хотя самого токена уже нет.

Теперь между аккаунтом и клиентом стоит класс *AccountAccess*. Выдаёт его только сам аккаунт, и только если он активен и токен не истёк:

```csharp
public AccountAccess? TryGetAccess() =>
    IsActive && !IsExpired
        ? new AccountAccess(Service, ExternalUserId, AccessToken, ExpiresAt)
        : null;
```

Фабрики и клиенты *SpotifyMusicService* и *YandexMusicService* теперь принимают только его:

```csharp
IMusicService Create(AccountAccess access);
```

Провайдер получает доступ уже после обновления токена:

```csharp
if (account.NeedsRefresh || account.IsExpired)
    await TryRefreshTokenAsync(account, service, cancellationToken);

var access = account.TryGetAccess()
             ?? throw new DomainException($"Токен доступа к {service} истёк, подключите аккаунт заново.");

return factory.Create(access);
```

Собрать клиента с отключённым аккаунтом или протухшим по нашим данным токеном теперь нельзя. «По нашим данным» тут главное, и эту границу стоит провести точно.

*TryGetAccess* проверяет две разные вещи. Первая — аккаунт активен в нашей базе. Вторая — токен не истёк по сохранённому *ExpiresAt*. Обе проверки локальные и делаются один раз, в момент создания снимка. Если срок не задан и *ExpiresAt* равен *null*, токен у нас вообще никогда не считается истёкшим.

Действителен ли токен у самого сервиса, *AccountAccess* не знает. Пользователь мог отозвать доступ в настройках Spotify, и мы узнаем об этом только по ответу на первый запрос.

Отдельно — время. Клиент получает токен один раз: *SpotifyMusicService* ставит заголовок *Authorization* в конструкторе. *ProcessTransferJob* берёт клиента в самом начале, поэтому весь перенос идёт с одним токеном. Если перенос длится дольше, чем живёт токен, тот истечёт прямо по ходу работы. Есть и пограничный случай: если обновление не удалось, но токен ещё не истёк, провайдер выдаст доступ с токеном, которому осталось меньше пяти минут — это *RefreshWindow*. Тип от всего этого не спасает, он гарантирует только, что на старте клиент получил активный аккаунт с неистёкшим по нашим данным токеном.

### Что получилось

Во всех трёх местах я убирал проверку «можно ли это сделать сейчас» и вместо неё давал тип, в котором ненужного метода нет.

Лучше всего typestate работает на коротких цепочках шагов внутри одной операции: OAuth и выдача доступа. Там каждый шаг выдаёт неизменяемый результат, который передаётся дальше и больше не меняется, и компилятор действительно держит порядок.

Слабее всего он там, где объект живёт долго, меняется на месте, сохраняется в базу и доступен нескольким процессам сразу. Это *TransferJob*. Там состояния — это обёртки над одним агрегатом. Они хорошо показывают, какие операции имеют смысл, и ловят ошибки в вызывающем коде, но устаревшую ссылку и чужое изменение в базе не видят. Без *unique* и линейных типов проверки внутри агрегата и охраняемая запись в хранилище остаются обязательными, а типы ложатся поверх них.
