## Три места, где я перебирал коллекцию в памяти вместо запроса

Пара слов про проект, иначе примеры непонятны.

MusicSync переносит плейлисты между стриминговыми сервисами: пользователь подключает Yandex Music и Spotify, выбирает плейлист, мы по каждому треку ищем соответствие в целевом сервисе и складываем найденное в новый плейлист. Сложность вся в матчинге — русскоязычная музыка приезжает то транслитом, то с *(feat. X)*, то другим релизом. Там, где алгоритм не уверен, совпадение помечается спорным и его разбирает человек.

.NET 10, PostgreSQL, Hangfire на фоновую обработку. ORM формально нет: EF Core остался только под `dotnet ef migrations`, в рантайме всё через Dapper. Переезд я сделал прошлым коммитом и считал вопрос закрытым.

Материал про ускорение кода фреймворков объясняет, почему не закрытым. Претензия там к привычке, которая остаётся после ORM: коллекция в базе воспринимается как список объектов, а раз список — по нему ходят циклом. Слои EF я убрал, а код в трёх главных сценариях переноса остался прежним. Сначала поднять все совпадения в память, потом отобрать нужные `foreach`'ем и LINQ'ом. Лежат они в Postgres, который умеет отбирать сам.

Ниже три таких места, замеры и баг, который я нашёл, пока их разбирал.

### Что внутри у ORM и что от неё осталось у меня

ORM у меня одна, EF Core, и та только для миграций. Конвейера у неё два, и они разные.

Чтение. LINQ-выражение сначала приводят к нормальному виду и раскрывают навигационные свойства. Потом из него строят дерево запроса: таблицы, `WHERE`, список колонок. По этому дереву отдельный генератор печатает текст SQL, а отдельный «шейпер» знает, как собрать объекты из того, что вернёт ридер. Готовый текст кешируется. Классы там с длинными именами — `QueryTranslationPreprocessor`, `QueryableMethodTranslatingExpressionVisitor`, `SelectExpression`, `QuerySqlGenerator`. Половина внутренние, между версиями переезжают.

Запись устроена иначе. Change tracker (`ChangeTracker.DetectChanges`) сравнивает объекты со снимком, снятым при загрузке. Разницу превращают в команду на каждую изменённую строку (`CommandBatchPreparer`, `ModificationCommand`), команды сортируют с оглядкой на внешние ключи, печатают INSERT/UPDATE (`UpdateSqlGenerator`) и складывают в батч (`ModificationCommandBatch`, у реляционных провайдеров — `ReaderModificationCommandBatch`). Батч уходит в базу за одно обращение, а не по обращению на строку. Сколько команд влезает — настройка `MaxBatchSize`.

Каким именно механизмом батч отправляется — одной командой через точку с запятой или через ADO.NET-батч (`DbBatch` / `NpgsqlBatch`), — зависит от версий EF Core и провайдера, и на своём проекте я это не проверял: EF у меня в рантайме не участвует. Мне отсюда нужно только то, что от версии не зависит: EF группирует изменения и не шлёт их по одному.

Вот эта вторая половина у меня и осталась, только написанная руками. Change Tracker'а у Dapper нет, вместо него в проекте `RegisterPersist`: репозиторий при загрузке агрегата кладёт в `UnitOfWork` делегат, а `SaveChangesAsync` прогоняет делегаты и коммитит.

```csharp
public async Task<TransferJob?> GetByIdAsync(Guid id, CancellationToken ct = default)
{
    // QueryMultiple: transfer_jobs + все track_matches по FK
    var aggregate = TransferJob.Load(...);
    foreach (var row in matches)
        aggregate.AddLoadedMatch(MapMatch(row));

    var insertedMatches = new HashSet<Guid>(aggregate.Matches.Select(m => m.Id));
    unitOfWork.RegisterPersist(async ct2 => await PersistChangesAsync(aggregate, insertedMatches, ct2));

    return aggregate;
}
```

ORM тут ни при чём, но точка входа в данные получилась такая же: одна, и поднимает граф целиком. Три места, где это было не нужно.

### 1. Решение пользователя по одному спорному совпадению

По смыслу тут вот что. Пользователь видит список неоднозначных совпадений и по каждому говорит: «вот этот трек» или «пропустить». Одно нажатие кнопки меняет одну строчку. Дополнительно надо понять, не последнее ли это спорное совпадение — если последнее, перенос пора досылать и закрывать.

Было так:

```csharp
private async Task<TransferJob> LoadAsync(Guid jobId, Guid userId, CancellationToken ct)
{
    var job = await transferJobs.GetByIdAsync(jobId, ct)
              ?? throw new DomainException("Перенос не найден.");

    if (job.UserId != userId)
        throw new DomainException("Перенос не найден.");

    return job;
}

var job = await LoadAsync(command.TransferJobId, command.UserId, ct);
job.ResolveManually(command.MatchId, command.ChosenTrackId);

if (job.AllTracksProcessed && !job.HasPendingReview)
    jobQueue.EnqueueTransferFinalization(job.Id);
```

Выглядит как три строчки. За ними: SELECT всех совпадений переноса (пятнадцать колонок на строку), построение `TrackMatch` и до двух `TrackIdentity` на каждую, `_matches.Find(m => m.Id == matchId)` линейным поиском по списку, `_matches.Exists(m => m.IsPendingReview)` ещё одним проходом, `_matches.Count` третьим. Для плейлиста на 500 треков — 500 строк по сети и до полутора тысяч объектов ради изменения одного поля.

Инварианты проверялись в трёх местах по очереди: владелец переноса — в хендлере, терминальность — в `EnsureNotTerminal`, «решение уже принято» — в `TrackMatch.EnsureResolvable`.

Стало — один запрос. Проверки поехали в `where`, пересчёт состояния в CTE:

```sql
with job as (
    select id, status, total_tracks from transfer_jobs
    where id = @Id and user_id = @UserId
),
target as (
    select m.id, m.status from track_matches m
    join job on job.id = m.transfer_job_id where m.id = @MatchId
),
decided as (
    update track_matches m
       set status = @NewStatus,
           matched_track_id = coalesce(@ChosenTrackId, m.matched_track_id),
           resolved_at = @Now
      from target, job
     where m.id = target.id
       and target.status in (2, 3)               -- NeedsReview, NotFound
       and job.status not in (3, 4, 5)           -- Completed, Failed, Cancelled
    returning m.id
),
counters as (
    select count(*)::int as processed,
           count(*) filter (where m.status = 2 and m.id <> @MatchId)::int as pending
    from track_matches m join job on job.id = m.transfer_job_id
),
resumed as (
    update transfer_jobs j set status = 1        -- обратно в Running
      from counters
     where j.id = @Id and j.status = 2 and counters.pending = 0
       and exists (select 1 from decided)
    returning j.id
)
select (select status from job) as job_status, ...
```

Числа тут для читаемости, в коде на их месте интерполяция из перечислений — `{(int)MatchStatus.NeedsReview}` и так далее, чтобы значения не разъехались с C#.

Хендлер стал таким:

```csharp
var resolution = await transferJobs.ApplyDecisionAsync(jobId, userId, decision, ct);

resolution.EnsureApplied();

if (resolution.ReadyToFinalize)
    jobQueue.EnqueueTransferFinalization(jobId);
```

`EnsureApplied` кидает те же `DomainException` с теми же текстами, что раньше кидал агрегат, только теперь по коду результата.

Одна тонкость, на которой я чуть не срезал угол. Data-modifying CTE в PostgreSQL работают на одном снимке данных: `counters` не увидит изменения, сделанного в `decided`, и посчитал бы только что решённое совпадение всё ещё спорным. Отсюда `and m.id <> @MatchId`: после этого запроса целевое совпадение спорным не будет в любом случае — сработал UPDATE или нет.

Вторая тонкость — `processed`. Хотелось заменить подсчёт строк на дешёвый `exists`, но тогда потерялось бы условие «все треки обработаны». Без него можно разобрать единственное спорное совпадение в середине матчинга, поставить финализацию, дослать половину плейлиста, а потом дослать его же целиком ещё раз. Дорогой счёт остался.

Замеры:

| Треков в переносе | Было | Стало |
|---|---|---|
| 200 | 7,2 мс | 5,2 мс |
| 500 | 9,7 мс | 5,5 мс |
| 2000 | 11,7 мс | 6,7 мс |
| 5000 | 35,7 мс | 8,2 мс |

В абсолютных числах скромно. Около пяти миллисекунд из «стало» — это фиксированные накладные на одну команду: создать scope, взять соединение из пула, `BEGIN`, `COMMIT`. От размера плейлиста они не зависят, и до двух тысяч треков за ними не видно разницы в объёме. На пяти тысячах они перестают быть главным слагаемым: «было» уходит в 35,7 мс, «стало» доходит до 8,2. `counters` тоже считает все строки переноса, но считает их внутри Postgres, без передачи и без создания объектов. План я не смотрел, `EXPLAIN` не гонял.

В колонке «было» к старому коду дописан UPDATE изменённой строки, которого там не было. Почему его там не было — ниже.

Когда я писал стенд, он упал на сверке: решений в базе оказалось 50 вместо 100. Половина — та, что прошла новым путём. Старый не записал ни одного. `PersistChangesAsync` умеет вставлять новые совпадения и обновлять саму строку переноса, а обновлять изменённое совпадение не умеет — такого кода там нет:

```csharp
var newMatches = job.Matches.Where(m => !insertedMatches.Contains(m.Id)).ToList();
if (newMatches.Count > 0)
    await InsertMatchesAsync(newMatches, ...);
```

`ResolveManually` честно менял объект в памяти, `SaveChangesAsync` честно коммитил транзакцию, API отвечал `204 No Content`, в `track_matches` не менялось ничего. Разбор спорных совпадений не работал вообще. Юнит-тесты на домен этого поймать не могли: они проверяют агрегат, а терялось всё между агрегатом и базой.

У EF в этом месте сработал бы change tracker: отслеживаемая сущность с изменённым свойством превращается в UPDATE сама. Выбросив ORM, я выбросил и это, а заметил только сейчас, когда первый раз прогнал этот путь по живой базе. В новом варианте теряться нечему: хендлер отправляет одну команду, и это UPDATE.

### 2. Досыл треков: какие совпадения доехали

По смыслу: матчинг закончен, спорное разобрано — надо сложить в целевой плейлист те треки, по которым есть принятое совпадение, в порядке исходного плейлиста.

Было дословно как в примере из материала — фильтр по условию плюс сортировка, оба проходом по коллекции в памяти:

```csharp
public IReadOnlyList<string> AcceptedTrackIds =>
    _matches.Where(m => m.IsAccepted)
            .OrderBy(m => m.Position)
            .Select(m => m.MatchedTrackId!)
            .ToList();
```

А чтобы до этого свойства добраться, Hangfire-задача поднимала агрегат целиком:

```csharp
var job = await transferJobs.GetByIdAsync(transferJobId, ct);
if (job is null || job.IsTerminal) return;

var destination = await musicServices.GetAsync(job.UserId, job.DestinationService, ct);
await FinalizeAsync(job, destination, ct);   // внутри — job.AcceptedTrackIds
```

Из всего агрегата ей нужны были четыре поля шапки и список строковых идентификаторов. Совпадения она поднимала, чтобы тут же их отфильтровать.

Стало то же самое словами SQL:

```sql
select matched_track_id
from track_matches
where transfer_job_id = @Id
  and status in (1, 4)                -- AutoAccepted, ManuallyResolved
  and matched_track_id is not null
order by position
```

И шапка отдельным однострочным запросом, без коллекции:

```csharp
var header = await transferJobs.GetHeaderAsync(transferJobId, ct);
if (header is null || header.IsTerminal || header.DestinationPlaylistId is null) return;

var destination = await musicServices.GetAsync(header.UserId, header.DestinationService, ct);

await AddAcceptedTracksAsync(transferJobId, header.DestinationPlaylistId, destination, ct);
var completion = await transferJobs.CompleteAsync(transferJobId, ct);
```

`CompleteAsync` — тоже один запрос: условия завершения («все треки обработаны», «спорных нет») проверяет сам UPDATE, так что между проверкой и записью ничего не успевает вклиниться. Раньше это были `if`'ы внутри `Complete()` над коллекцией, загруженной неизвестно когда.

Замеры:

| Треков в переносе | Было | Стало |
|---|---|---|
| 200 | 3,1 мс | 1,5 мс |
| 500 | 11,6 мс | 2,0 мс |
| 2000 | 14,9 мс | 3,0 мс |
| 5000 | 44,4 мс | 6,4 мс |

Разница ожидаемая: по сети едет одна строковая колонка вместо пятнадцати, и не создаётся ни одного объекта, кроме самих строк. Коммита нет ни в одном из вариантов, сценарий только читает. Строка на 500 шумная: в одном из трёх прогонов «было» вышло 4,6 мс вместо примерно двенадцати. Почему — не разбирался, в таблице медиана.

Заплатил я за это дублированием. Правило «какие треки считаются доехавшими» теперь записано дважды — в `AcceptedTrackIds` на C# и в тексте запроса. Одна из копий строка, компилятор в неё не заглядывает. Поменяю условие в одном месте, второе останется старым, и узнаю я об этом от пользователя, которому в плейлист приехало не то.

Свойство в домене я оставил как эталон, тесты на него тоже, а стенд перед замерами сверяет два определения:

```csharp
Require(viaAggregate.SequenceEqual(viaSql),
    $"Проекция разошлась с агрегатом: {viaSql.Count} против {viaAggregate.Count}");
```

Правильное место для такой проверки — интеграционный тест, а не стенд. Его пока нет.

### 3. Запись батча совпадений

По смыслу: фоновая обработка идёт батчами по 25 треков — нашли соответствия, записали, зафиксировали прогресс, поехали дальше. Батчи нужны, чтобы перенос на тысячу треков не держал одну транзакцию десять минут и чтобы при падении не терялось всё сразу.

Было:

```csharp
var parameters = matches.Select(m => new
{
    m.Id, m.TransferJobId, m.Position, m.SourceTrackId, m.MatchedTrackId,
    Status = (int)m.Status, m.MatchedAt, m.ResolvedAt,
    Confidence = m.Confidence.Value,
    SourceArtist = m.SourceTrack.Artist, /* ... */
});

await connection.ExecuteAsync(new CommandDefinition(sql, parameters, transaction, ct));
```

Выглядит как одна вставка списка. Это тот случай, про который в материале сказано «учитывайте внутреннюю механику»: перегрузка `ExecuteAsync`, которой скормили последовательность параметров, выполняет команду по разу на элемент — она для того и сделана. Двадцать пять обращений к серверу на батч, пятьсот на плейлист из пятисот треков. Замер подтверждает: разрыв на двадцати пяти строках — цифры ниже — объясняется количеством обращений, на самой вставке столько не теряется.

Скорее всего, EF Core здесь был бы быстрее: изменения он группирует, а я написал цикл. Проверять не стал, EF в рантайме у меня нет, так что это рассуждение, а не замер. Направление всё равно понятное — я ушёл с ORM ради производительности и написал в этом месте хуже, чем делает ORM.

Стало — колонки уезжают массивами и разворачиваются в строки на стороне Postgres:

```sql
insert into track_matches (id, transfer_job_id, position, ...)
select * from unnest(
    @Id::uuid[], @TransferJobId::uuid[], @Position::int[],
    @SourceTrackId::text[], @MatchedTrackId::text[],
    @Status::int[], @MatchedAt::timestamptz[], @ResolvedAt::timestamptz[],
    @Confidence::float8[],
    @SourceArtist::text[], @SourceTitle::text[], @SourceDurationMs::int[],
    @MatchedArtist::text[], @MatchedTitle::text[], @MatchedDurationMs::int[])
```

Массивы пришлось передавать мимо Dapper: `IEnumerable`-параметр он разворачивает в список `@p1, @p2, ...` — удобно для `IN`, но мне нужен массив одним параметром.

```csharp
await using var command = connection.CreateCommand();
command.CommandText = sql;
command.Transaction = transaction;

AddArray(command, "Id", matches.Select(m => m.Id).ToArray());
AddArray(command, "Position", matches.Select(m => m.Position).ToArray());
// ... по колонке на массив

await command.ExecuteNonQueryAsync(cancellationToken);
```

Замеры — время на один батч из 25 строк, двадцать батчей подряд:

| | Было | Стало |
|---|---|---|
| 25 строк × 20 батчей | 56–112 мс | 3,4–6,1 мс |

Разброс внутри диапазонов — это разные прогоны при разном наполнении самой `track_matches`. Отношение по прогонам от 13 до 26 раз, медиана около восемнадцати. И это на локальной базе, где обращение к серверу стоит недорого; на базе с более длинным сетевым путём разрыв станет больше, потому что у старого варианта обращений в двадцать пять раз больше и каждое подорожает.

Выигрыш тут от количества обращений; в двух предыдущих случаях он был от объёма данных. Переносить в SQL по смыслу ничего не пришлось — запрос делает то же самое, что и раньше, за один раз.

### Про упорядоченность

Третий пункт материала — про то, что порядок результата легко принять за случайность. У меня два таких места, и они разные.

В `GetAcceptedTrackIdsAsync` `order by position` обязателен. Плейлист должен выглядеть как исходный, порядок — требование, и оно записано тестом `AcceptedTrackIds_KeepsSourceOrder`. Раньше его обеспечивал `.OrderBy(m => m.Position)` в LINQ; если бы я переписал запрос без `order by`, ничего бы не упало — строки вставлялись в порядке позиций, и Postgres, скорее всего, вернул бы их примерно в том же порядке. До первой смены плана или переупаковки таблицы.

А вот в `ConnectedAccountRepository.GetAllAsync` есть `order by connected_at desc`, и зачем он там — я не знаю. Список подключённых аккаунтов у пользователя длиной два. Скорее всего порядок туда попал потому, что «пусть свежие сверху». Проверить это по коду нельзя: на него никто не закладывается явно, но и не заявлено, что закладываться нельзя. Материал предлагает лечить такое типом результата — возвращать множество там, где порядок не обещан. Я пока отметил это для себя.

### Что осталось

Загрузка агрегата ради одного поля. В `ProcessTransferJob`, в обработчике ошибки:

```csharp
var job = await transferJobs.GetByIdAsync(transferJobId, CancellationToken.None);
job?.Fail(ex.Message);
```

Поднимаются все совпадения, чтобы записать `status` и `failure_reason`. Тот же случай, что первый, лечится так же — но это путь обработки сбоя, где лишние сто миллисекунд никого не волнуют, и трогать его вместе с остальным я не стал.

Чтение, которое пишет. `StartTransferHandler` перед запуском переноса проверяет, что оба аккаунта подключены:

```csharp
var account = await accounts.GetAsync(userId, service, ct);
if (account is null || !account.IsActive) throw new DomainException(...);
```

`GetAsync` при каждой загрузке регистрирует persist-делегат, а `SaveChangesAsync` делегаты не очищает — их чистит только откат. Чтение двух аккаунтов оборачивается двумя UPDATE'ами `connected_accounts`, и потом ещё по два на каждый коммит батча внутри переноса, потому что `MusicServiceProvider` ходит за аккаунтом через тот же репозиторий. Колонки токенов там `varchar(4096)`. Чинится это разделением репозитория на «загрузить для изменения» и «загрузить посмотреть», и это уже не точечная правка.

Квадратичная проверка в `RecordMatch`:

```csharp
if (_matches.Exists(m => m.SourceTrackId == sourceTrackId))
    throw new DomainException($"Трек {sourceTrackId} уже обработан.");
```

На каждый из N треков — проход по уже записанным. В базе на `(transfer_job_id, source_track_id)` висит уникальный индекс, который то же самое гарантирует надёжнее. Здесь я сознательно ничего не делаю: это инвариант агрегата, он должен ломаться доменным исключением, а не нарушением ограничения из Npgsql. Заметно это станет тысячах на десяти треков, которых в MVP не будет.

И то, что вылезло по дороге. `GetByIdAsync` вообще не работал на живой базе. Npgsql отдаёт `timestamp with time zone` как `DateTime`, а строки репозитория объявлены через `DateTimeOffset`, и Dapper на этом падал:

```
A parameterless default constructor or one matching signature ... is required
```

Починил обработчиком типа, на стенде это подтвердилось: после его регистрации Dapper подбирает конструктор записи нормально.

```csharp
internal sealed class DateTimeOffsetTypeHandler : SqlMapper.TypeHandler<DateTimeOffset>
{
    public override DateTimeOffset Parse(object value) => value switch
    {
        DateTimeOffset offset => offset,
        DateTime time => new DateTimeOffset(DateTime.SpecifyKind(time, DateTimeKind.Utc)),
        _ => throw new InvalidCastException(...)
    };
}
```

DTO запросов (`TransferStatusDto`, `PendingMatchDto`) объявлены так же, через `DateTimeOffset`, так что проблема у них, по идее, была та же. Проверить не могу: хендлеры запросов стенд не дёргает, руками я их не гонял.

Оба дефекта лежали в коде с прошлого коммита. Нашлись на первом прогоне по настоящему Postgres — интеграционных тестов в проекте нет, до стенда этот код по живой базе никто не гонял.

### Как считались замеры

Стенд — `bench/MusicSync.Bench`, Postgres в Docker на той же машине, Windows. Одна прогревочная итерация, дальше среднее по 49; в таблицах — медиана трёх таких прогонов.

Случаи 1 и 2 меряются целиком, так, как их дёргает приложение: создание DI-scope, соединение из пула, `BEGIN`, сами запросы. В случае 1 к этому добавляется `SaveChangesAsync` — он прогоняет persist-делегаты, делает `COMMIT` и сразу открывает следующую транзакцию. В случае 2 коммита нет, это чистое чтение: scope в конце отпускает транзакцию и возвращает соединение в пул. Случай 3 меряется без всей этой обвязки: соединение открыто снаружи цикла, транзакция не заводится, внутрь попадает только сама вставка батча.

Отсюда разные «полы» у чисел. В случае 2 «стало» не опускается ниже примерно 1,5 мс, в случае 1 — ниже примерно 5 мс. Разница между ними — это `COMMIT`, открытие следующей транзакции и более тяжёлый запрос; по частям я это не разбирал. Сеть тут короткая, но не бесплатная: Postgres в Docker на Windows берёт за обращение заметно больше, чем локальный сокет, так что пол завышен.

Ещё оговорка к случаю 3: он гоняется после того, как в таблице уже лежит засеянный перенос на N строк. Поэтому его абсолютные числа между разными N сравнивать нельзя — сравнимы только «было» и «стало» внутри одного прогона.
