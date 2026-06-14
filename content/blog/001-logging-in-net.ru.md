+++
title = "Logging in .NET"
date = 2024-08-28
description = "Структурированное логгирование в .NET и best-practices"
+++

> Изначально этот текст был написан для внутреннего доклада. Спустя какое то время я решил его опубликовать в качестве первого поста.

## Введение   
Существует достаточно много различных библиотек для логирования - nlog, serilog, log4n, elmah и т.д., но здесь будет рассматриваться использование стандартного Microsoft.Extensions.Logging, т.к. его возможностей, в целом, достаточно.   
Также хотелось бы сразу призвать в целом писать логи. Необязательно логировать каждое действие и каждую строчку кода, но ошибки, нестандартное поведение и основные действия бизнес-логики стоит писать.   
Также логи нужны не только вам, разработчикам сервиса, но и:

- другим разработчикам, для понимания “а что же происходит в том сервисе”   
- при тестировании для более корректного составления задачи (проще же когда приносят ссылку на лог, чем абстрактное “при нажатии кнопки оно ломается”)   
- поддержке для разбора ошибки/инцидента   

## Уровни логгирования   
Почему это вообще важно?

- Если писать все логи (и важные, и какие то технические) как, например, `Information` , то разбираться потом в этой куче уже не захочется   
- Отсутствие возможности адекватно фильтровать логи для записи в elasticsearch на проде

|Level | Description |
|:-----|:------------|
|Trace | Наиболее низкий уровень логирования для отслеживания всех происходящих событий |
|Debug | Сообщения которые могут быть полезны для понимания поведения приложения |
| Information | Основные сообщения |
| Warning |Ситуации, которые, в теории, не должны происходить, однако корректно обрабатываются |
|Error | Некритичные ошибки, которые не были предусмотрены |
| Critical | Критические ошибки, при возникновении которых приложение не может продолжать работу |
| None | Используется только в фильтрах для отключения логов |

[Link to docs](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging#log-level)

## Обработка ошибок   
Перед тем как говорить о предметах для логгирования, думаю стоит поговорить об обработке ошибок. Ковыряя BEAM-машину и Erlang я нашел интересный принцип “let it fail”. Суть такова, что не надо ловить эксепшен, если вы не собираетесь с ним ничего делать, пускай он докатится до обработчика, который его ожидает. Например часто можно встретить подобный код:

```
try
{
    DoSomething();
}
catch (Exception ex)
{
    _logger.LogWarning(ex.ToString());
    throw ex;
}

```

В чем проблема?

- Засорение методов по своей сути бесполезным кодом, который только ухудшает читаемость   
- Потеря стактрейса при перепрокидывании исключения.   
- Вызывая какой то метод, который кидает исключение можно ли угадать, не заглядывая в него, залогировал ли он его? В итоге можно получить в логах несколько записей об одном и том же исключении по сути

![Перекидывание исключения](/assets/logging-in-net/exceptions-rethrow-ladder.png)

### Что тогда делать?   
Если не собираетесь ничего делать с исключением, то незачем и ловить его, не превращайте все это в Go с бесконечным перекидыванием ошибок по стеку. Но удостоверьтесь, что оно все же кем-то будет поймано, например каким-то глобальным обработчиком, как `IExceptionFilter`   
Также стоит разделять системные и доменные ошибки, т.е. не стоит на все ошибки возвращать 500 Internal server Error. В качестве примера возьмем такую ситуацию, что пользователь хочет добавить какого-то питомца, но имя уже занятно.   
Соответственно создаем корневой тип исключения для доменных ошибок:   
```
public abstract class DomainException(string message, Exception? innerException = null)
    : Exception(message, innerException)
{
    public abstract string ERROR_CODE { get; }

    public virtual ProblemDetails GetProblemDetails() => new()
    {
        Title = ERROR_CODE,
        Detail = message,
        Status = (int)HttpStatusCode.UnprocessableEntity
    };
}

```
Создаем конкретный тип для исключения с понятным текстом ошибки:   
```
public class NickTakenException(string nickName, Exception? innerException = null) 
    : DomainException($"Nick {nickName} already taken", innerException)
{
    public override string ERROR_CODE => "NICKNAME_TAKEN";
}

```
Соответственно теперь оборачиваем исключение в доменное:   
```
try
{
    await _petsRepository.Create(pet);
}
catch (UniqueConstraintException ex)
{
    throw new NickTakenException(pet.NickName, ex);
}

```
А в глобальном обработчике уже разделяем логику для доменных исключений и остальных:   
```
public class ExceptionFilter(ILogger<ExceptionFilter> logger) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, 
        EndpointFilterDelegate next)
    {
        try
        {
            return await next(context);
        }
        catch (DomainException e)
        {
            logger.LogInformation(
                eventId: ApplicationEventId.UnprocessibleRequest,
                message: "Cannot process request due to domain exception: {exception}",
                e.Message);

            return TypedResults.Problem(e.GetProblemDetails());
        }
        catch (Exception e)
        {
            logger.LogError(
                eventId: ApplicationEventId.RequestFailed,
                exception: e,
                message: "Request failed"
            );
            
            throw;
        }
    }
}

```
Т.е. получается иерархия исключений:   

![Exception hierarchy](/assets/logging-in-net/exceptions-hierarchy.png)

## А что логгировать?   
1. Исключения. Какие и как разобрались   
2. Доменные события (мутация данных, действия пользователя)   
3. Технические данные для возможного дебага (с LogLevel.Debug или LogLevel.Trace). Слишком подробное логирование на проде будет тормозить как приложение, таки elasticsearch.   
   
## Формат логов   
### Плоское логгирование   
Исторически логи представляли собой просто какое то текстовое сообщение (которое писалось в файл или stdout). Но это приводит к следующей проблеме - допустим есть сообщение следующего вида:   
```
[2022-01-20 15:45:27] ERROR: Failed to update entity with ID 1234: Database constraint violation - Duplicate entry 'example@example.com' for key 'email_unique'.

```
Мы хотим достать все сообщения об ошибках с сущностью у которой ID = 1234:   
```
$ cat test.log | grep 'ERROR' | grep 'ID 1234'
[2022-01-20 15:45:27] ERROR: Failed to update entity with ID 1234: Database constraint violation - Duplicate entry 'example@example.com' for key 'email_unique'.

```
Но в тестовом файле с логами также была строчка, которая не попала в выборку, потому что формат *чуточку* другой:   
```
[2022-01-20 15:45:20] ERROR: Failed to deserialize response for get entities with ID=1234

```
В общем, думаю суть понятна, попытки грепать или как то иначе искать логи в неструктурированном тексте приводят упусканию записей, которые могли бы оказаться полезными и при этом попадение в выборку *левых* данных, которые только отвлекают и сбивaют.   
### Структурированное логирование   
Было бы удобно искать логи в схожей манере с записями в БД через SQL?

![Структуирование логов](/assets/logging-in-net/structurize-log.png)

```
{
    "timestamp": "2022-01-20 15:45:27",
    "severity": "error",
    "messageTemplate": "Failed to update entity with ID {entityId}: {exceptionMessage}.",
    "messageDisplay": "Failed to update entity with ID 1234: Database constraint violation - Duplicate entry 'example@example.com' for key 'email_unique'.",
    "data": {
        "entityId": 1234,
        "exceptionMessage": "Database constraint violation - Duplicate entry 'example@example.com' for key 'email_unique'"
    }
}

```
И соответственно теперь для поиска всех ошибок связанных с id у entity 1234 можно найти простым запросом (пример elasticsearch):   
```
severity: error and data.entityId: 1234

```
Но при этом в большинстве случаев логгирование происходит следующим образом:   
```
public async Task<Pet> CreatePet(Pet pet)
{
	_logger.LogInformation($"Creating pet with name {pet.NickName}");

	// collapsed
}

```
В том числе так:   
```
logger.LogInformation($"Got response: {JsonSerializer.Serialize(response)}");

```
Оба этих варианта демонстируют плохие практики, т.к. в итоге дают не структурированные логи, а плоские записи. Дальнейшая попытка поиска по таким логам приводит к полнотекстовому поиску по подстроке (запрос ecs):   
```
FullText: *some value*

```
Подобное наследует проблемы, характерные для плоских логов и при этом добавляет лишнюю нагрузку для elasticsearch.   
Корректный вариант:   
```
_logger.LogInformation("Creating pet with name {NickName}", pet.NickName);

```
Соответственно **не надо** интерполировать строки при логировании и тем более сериализовать данные в сообщение.   
> При этом надо отметить, что имя которое вы указываете для значения в шаблоне будет использоваться для записи структурированных данных в хранилище (т.е. elasticsearch) и не обязано коррелировать с именем переменной / поля / свойства, которое передается аргументов. Placeholder в шаблоне коррелирует со значением из структурированной части по очередности, а не имени.   

[Link to docs](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging#log-message-template)   
## EventId   
Ок, логи пишутся в структурированном виде. Но как теперь найти все логи в импровизированном приложении, когда создается запись о питомце? Опять искать по входам подстроки? Для этого как раз пригодятся [EventId](https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.logging.eventid?view=dotnet-plat-ext-8.0). Соответственно для управления ими из одного места создаем подобный класс:   
```
public static class ApplicationEventId
{
    public const int CreatePet = 1_000;
}

```
И теперь при создании питомца:   
```
public async Task<Pet> CreatePet(Pet pet)
{
    logger.LogInformation(
        eventId: ApplicationEventId.CreatePet,
        message: "Creating pet with nickname({nickName})", 
        args: pet.NickName
    );
    
    ...
}

```
Соответственно это позволяет теперь искать логи по EventId:

![Поиск по EventId](/assets/logging-in-net/elastic-eventid.png)

В реализации Elastic.Extensions.Logging EventId записывается как: 
`{EventId.Name}({EventId.Id})`

## Интерфейсы логгирования   
### Стандартный интерфейс   
```
logger.LogInformation(
    eventId: ApplicationEventId.CreatePet,
    message: "Creating pet with nickname({nickName})", 
    args: pet.NickName
);

```
### Source-generated loggers   
Логирование стандартным образом может очень сильно загромождать код и ухудшать его читаемость. К примеру есть следующий код для сервиса кеширования:   
```
public class CacheService<TKey, TValue>
    : IDisposable
    where TKey : IEquatable<TKey>
    where TValue : class
{
    private sealed record CacheWrapper(DateTime ExpiresAt, TValue Value);

    private readonly ConcurrentDictionary<TKey, CacheWrapper> _cache = new();
    private readonly TimeSpan _timeToLive = TimeSpan.FromMinutes(10);
    private readonly TimeSpan _purgingInterval = TimeSpan.FromSeconds(10);
    private readonly Timer _timer;

    public CacheService()
    {
        _timer = new Timer(_purgingInterval);
        _timer.Elapsed += PurgeExpired;
        
        _timer.Start();
    }

    private void PurgeExpired(object? sender, ElapsedEventArgs e)
    {
        if (_cache.IsEmpty)
            return;

        var expiredKeys = _cache
            .Where(kv => kv.Value.ExpiresAt < DateTime.UtcNow)
            .Select(kv => kv.Key)
            .ToArray();

        foreach (var key in expiredKeys)
        {
            _cache.Remove(key, out _);
        }
    }
    
    public Task<TValue?> Get(TKey key, Func<TKey, Task<TValue?>> retrieve)
        => _cache.GetValueOrDefault(key) switch
        {
            { } cachedValue when cachedValue.ExpiresAt > DateTime.UtcNow => Task.FromResult<TValue?>(cachedValue.Value),
            null => GetAndSet(key, retrieve),
            _ => GetAndSetIfExpired(key, retrieve),
        };

    private async Task<TValue?> GetAndSet(TKey key, Func<TKey, Task<TValue?>> retrieve)
    {
        var value = await retrieve(key);

        if (value is null)
            return null;

        _cache.TryAdd(key, new CacheWrapper(DateTime.UtcNow + _timeToLive, value));

        return value;
    }

    private Task<TValue?> GetAndSetIfExpired(TKey key, Func<TKey, Task<TValue?>> retrieve)
    {
        _cache.Remove(key, out _);

        return GetAndSet(key, retrieve);
    }

    public void Dispose() => _timer.Dispose();
}

```
Теперь добавим некоторое логгирование на примере одного метода (слева его первоначальный вид, справа - с логами):   
```
private async Task<TValue?> GetAndSet(TKey key, Func<TKey, Task<TValue?>> retrieve)
{
    var value = await retrieve(key);

    if (value is null)
        return null;

    _cache.TryAdd(key, new CacheWrapper(DateTime.UtcNow + _timeToLive, value));

    return value;
}

private async Task<TValue?> GetAndSet(TKey key, Func<TKey, Task<TValue?>> retrieve)
{
    _logger.LogDebug(
        InternalEventId.CacheValueMissed,
        "Missed cache value for key {key}",
        key
    );
    
    var value = await retrieve(key);

    if (value is null)
    {
        _logger.LogDebug(
            InternalEventId.CacheRetrievedNullActualValue,
            "Cache retrieved null actual value for key {key}",
            key
        );
        
        return null;
    }

    _cache.TryAdd(key, new CacheWrapper(DateTime.UtcNow + _timeToLive, value));
    
    _logger.LogDebug(
        InternalEventId.CacheReturnedActualValue,
        "Cache returned actual value {value} for key {key}",
        value,
        key
    );

    return value;
}

```
Маленький и компактный метод вырос в 2 раза, и читать его уже не так приятно. Можно было бы писать логи в одну строчку, но это меняет только направление роста метода (вертикально или горизонтально).   
В NET8 появилась фича “Compile-time logging source generation\*\*”.\*\* Рассмотрим алгоритм работы на примере первого сообщения логгирования из приведенного выше примера:   
```
internal static partial class CacheServiceLogger
{
    [LoggerMessage(
        EventId = InternalEventId.CacheValueMissed,
        Level = LogLevel.Debug,
        Message = "Missed cache value for key {key}"
    )]
    internal static partial void LogCacheMiss(this ILogger logger, object key);
}

```
Собственно здесь указывается то же самое, что и ранее при прямом вызове метода, но при этом генерируется во что то подобное:   
```
    partial class CacheServiceLogger
    {
        [global::System.CodeDom.Compiler.GeneratedCodeAttribute("Microsoft.Extensions.Logging.Generators", "8.0.9.3103")]
        private static readonly global::System.Action<global::Microsoft.Extensions.Logging.ILogger, global::System.Object, global::System.Exception?> __LogCacheMissCallback =
            global::Microsoft.Extensions.Logging.LoggerMessage.Define<global::System.Object>(global::Microsoft.Extensions.Logging.LogLevel.Debug, new global::Microsoft.Extensions.Logging.EventId(2010, nameof(LogCacheMiss)), "Missed cache value for key {key}", new global::Microsoft.Extensions.Logging.LogDefineOptions() { SkipEnabledCheck = true }); 

        [global::System.CodeDom.Compiler.GeneratedCodeAttribute("Microsoft.Extensions.Logging.Generators", "8.0.9.3103")]
        internal static partial void LogCacheMiss(this global::Microsoft.Extensions.Logging.ILogger logger, global::System.Object key)
        {
            if (logger.IsEnabled(global::Microsoft.Extensions.Logging.LogLevel.Debug))
            {
                __LogCacheMissCallback(logger, key, null);
            }
        }
    }

```
А в самом методе теперь достаточно просто вызвать метод: `_logger.LogCacheMiss(key);`   
В итоге использовав source-generated логгеры получается следующий вид метода (слева версия с source-generated логами, справа предыдущая):   
```
private async Task<TValue?> GetAndSet(TKey key, Func<TKey, Task<TValue?>> retrieve)
{
    _logger.LogCacheMiss(key);
    
    var value = await retrieve(key);

    if (value is null)
    {
        _logger.LogCacheNullActualValue(key);
        return null;
    }

    _cache.TryAdd(key, new CacheWrapper(DateTime.UtcNow + _timeToLive, value));
    
    _logger.LogActualValue(value, key);

    return value;
}

private async Task<TValue?> GetAndSet(TKey key, Func<TKey, Task<TValue?>> retrieve)
{
    _logger.LogDebug(
        InternalEventId.CacheValueMissed,
        "Missed cache value for key {key}",
        key
    );
    
    var value = await retrieve(key);

    if (value is null)
    {
        _logger.LogDebug(
            InternalEventId.CacheRetrievedNullActualValue,
            "Cache retrieved null actual value for key {key}",
            key
        );
        
        return null;
    }

    _cache.TryAdd(key, new CacheWrapper(DateTime.UtcNow + _timeToLive, value));
    
    _logger.LogDebug(
        InternalEventId.CacheReturnedActualValue,
        "Cache returned actual value {value} for key {key}",
        value,
        key
    );

    return value;
}

```
Скорость логгирования (BenchmarkDotnet): 

| Method |      Mean |         Error |     StdDev |
|:-----------|:--------------|:--------------|:--------------|
|  NoLog | 0.0103 ns | 0.0015 ns | 0.0013 ns |
| StandardLog | 10,037.8848 ns | 199.7660 ns | 222.0394 ns |
| SourceGeneratedLog |  9,832.4832 ns | 196.4297 ns | 557.2383 ns |
|        SuppressLog |     42.0594 ns |   0.5636 ns |   0.4706 ns |

Crank (bombardier):

| Bench name | First Request (ms) |      Requests | Bad responses | Latency 50th (ms) | Latency 75th (ms) | Latency 90th (ms) | Latency 95th (ms) | Latency 99th (ms) | Mean latency (ms) | Max latency (ms) | Requests/sec | Requests/sec (max) | Read throughput (MB/s) |
|:-----------|:-------------------|:--------------|:--------------|:------------------|:------------------|:------------------|:------------------|:------------------|:------------------|:-----------------|:-------------|:-------------------|:-----------------------|
|      NoLog |                681 | **2,873,553** |             0 |              1.01 |              1.59 |              2.01 |              2.25 |              3.52 |          **1.33** |            73.50 |  **191,056** |            226,472 |                  35.80 |
|     StdLog |                675 |   **133,919** |             0 |             29.45 |             33.71 |             38.27 |             42.16 |             52.63 |         **28.69** |           156.17 |    **9,079** |             51,009 |                   1.67 |
|  SourceGen |                684 |   **133,178** |             0 |             29.54 |             34.23 |             39.23 |             42.97 |             52.21 |         **28.85** |           125.06 |    **8,990** |             67,469 |                   1.66 |
| DefinedLog |                668 |   **133,342** |             0 |             29.77 |             33.53 |             37.77 |             41.66 |             53.54 |         **28.81** |           170.11 |    **9,010** |             64,246 |                   1.66 |
| SupressLog |                685 | **2,837,691** |             0 |              1.01 |              1.64 |              2.02 |              2.31 |              3.54 |          **1.35** |            94.60 |  **188,341** |            216,767 |                  35.35 |

| Bench name |      Requests | Mean latency (ms) | Requests/sec |
|:-----------|:--------------|:------------------|:-------------|
|      NoLog | **2,873,553** |          **1.33** |  **191,056** |
|     StdLog |   **133,919** |         **28.69** |    **9,079** |
|  SourceGen |   **133,178** |         **28.85** |    **8,990** |
| DefinedLog |   **133,342** |         **28.81** |    **9,010** |
| SupressLog | **2,837,691** |          **1.35** |  **188,341** |

Microsoft позиционирует source-generated логи как High-Performance решение, но, как показывают тесты, особой разницы в скорости между различными способами нет. Параллельно также к подобному выводу пришел [Кирилл Бажайкин](https://t.me/csharp_gepard/79)   
### Структурированное логгирование из Elastic.Extensions.Logging   
```
_logger.CreateEntry(LogLeve.Information, ApplicationEventId.CreatePet)
    .Message($"Creating pet with nickname({pet.NickName})")
    .Data(new { NickName = pet.NickName })
    .Write()
    ;

```
В целом ничто не мешает использовать и стандартное логгирование и этот способ как расширения для более красивого вызова:   
```
public static void LogCreatePet(this ILogger<PetService> logger, string nickName) => _logger
    .CreateEntry(LogLeve.Information, ApplicationEventId.CreatePet)
    .Message($"Creating pet with nickname({nickName})")
    .Data(new { NickName = nickName })
    .Write()
    ;

```
## Консоль на проде   
Для сравнения ниже приведен бенчмарк логгирования в консоль и в файл. Для логгирования в файл использовался `NReco.Logging.File`   
| **Method** |     **Mean** |   **Error** |  **StdDev** |   **Median** |
|:-----------|:-------------|:------------|:------------|:-------------|
|    Console | 232,008.7 ns | 3,605.41 ns | 3,372.50 ns | 231,885.4 ns |
|       File |     650.7 ns |    23.81 ns |    70.22 ns |     619.7 ns |

Из цифр видно что логирование в консоль *в разы* медленее. Но тут надо понимать что это данные синтетического бенчмарка и в реальном приложении первый же поход в БД или другой сервис сводит разницу на ноль. Но все же, зачем логировать в консоль если это пустая трата ресурсов?   
## Конфигурирование   
Можно фильтровать логи для записи по их уровню и категории:   
```
builder.Logging.AddFilter<ConsoleLoggerProvider>("Default", LogLevel.Critical);
builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Default", LogLevel.Information);
builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Microsoft", LogLevel.Warning);
builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Jaeger", LogLevel.Warning);

```
Но тогда эти правила будут применяться для всех контуров, а на деве/стейдже хотелось бы иметь больше информации и тогда начинается такая лапша:   
```
if (builder.Environment.IsProduction()) 
{
 builder.Logging.AddFilter<ConsoleLoggerProvider>("Default", LogLevel.Critical);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Default", LogLevel.Information);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Microsoft", LogLevel.Warning);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Jaeger", LogLevel.Warning);
}
else
{
 builder.Logging.AddFilter<ConsoleLoggerProvider>("Default", LogLevel.Information);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Default", LogLevel.Information);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Microsoft", LogLevel.Information);
 builder.Logging.AddFilter<Elastic.Extensions.Logging.Provider>("Jaeger", LogLevel.Warning);
}

```
Но зачем расписывать подобное в коде, если, по сути, это конфиг, который можно прекрасно запихнуть в консул, настроив для каждого контура как угодно и при этом обновления будут работать “на лету”.   
Для конфигурирования логгирования предлагается подобный json:   
```
{
    "Logging": {
        "Console": {
            "LogLevel": {
                "Default": "Critical"
            }
        },
        "Elastic": {
            "Default": "Information",
            "Microsoft": "Warning",
            "Jaeger": "Warning"
        }
    }
}

```
Что соответсвенно можно представить как такой набор пар ключ-значение в консуле:   

![Конфи логирования в Consul](/assets/logging-in-net/logging-config-consul.png)

Соответственно в данном примере заглушен вывод логов в консоль на проде (только критичные ошибки), в то время как в эластику пишутся пишутся все логи c Level ≥ Information, кроме `Microsoft` и `Jaeger` у которых пишутся только Warning и выше.

## Заключение   
1. Пишите логи, они помогут и вам, и другим;   
2. Пишите их осознанно, с понятными сообщениями и контекстом, проставляйте категорию и выбирайте правильный уровень;   
3. Не используйте строковую интерполяцию и, тем более, JsonSerializer, пишите их структурированно;   
4. Помимо того чтобы проставлять уровни для логов, настраивайте фильтрацию ненужных логов для контуров;   
5. Обрабатывайте корректно ошибки, не лепите бесполезные try-catch\`и.   

