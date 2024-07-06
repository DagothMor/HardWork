# 1. При подписании договора добавить не только поле подписанта, но и скрытое поле его замещающих:

```csharp
public class Contract
{
    public int Id { get; set; }
    public string Signer { get; set; }
    private List<string> Substitutes { get; set; } = new List<string>();

    public void Sign(string signer, List<string> substitutes)
    {
        Signer = signer;
        Substitutes = substitutes;
    }
}
```
Добавляем скрытое поле замещающих, тогда отлаживать бизнес процессы с учетом отпусков итд будет проще, понятно сразу, кто может заместить человека на данный момент.
# 2. При получении договора сразу же добавить список гуишников доп соглашений:


```csharp
public class Contract
{
    public int Id { get; set; }
    public string Signer { get; set; }
    public List<string> Substitutes { get; set; } = new List<string>();
    private List<string> AdditionalAgreements { get; set; } = new List<string>();

    public void AddAdditionalAgreement(string agreementId)
    {
        AdditionalAgreements.Add(agreementId);
    }
}
```
Так же добавляем скрытое поле, состоящее лишь из гуидов доп соглашений, для явного анализа аналитиками.

# 3. Добавить глобальный кеш истории бизнес шагов в рамках бизнес процесса:

Простенький пример
```csharp
using Microsoft.Extensions.Caching.Memory;

public class BusinessProcess
{
    private readonly IMemoryCache _cache;

    public BusinessProcess(IMemoryCache cache)
    {
        _cache = cache;
    }

    public void ExecuteStep(string stepName, object stepData)
    {
        try
        {
            // здесь реализуем логику выполнения шага бизнес-процесса
            // сохраняем результат выполнения шага в кеш
            _cache.Set($"BusinessProcess:Step:{stepName}", stepData);
        }
        catch (Exception ex)
        {
            // здесь реализуем логику обработки ошибок
            // сохраняем информацию об ошибке в кеш
            _cache.Set($"BusinessProcess:Error:{stepName}", ex.Message);
        }
    }
}
```
Таким образом при возникновении ошибки в определенном шаге(исключения), или при логической ошибке, аналитик сможет зайти в таблицу бд(или в логах прометеуса) и увидеть пошагово какой результат был в каждом бизнес шаге.

