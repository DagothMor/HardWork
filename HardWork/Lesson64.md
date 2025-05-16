Функциональный интерфейс
```cs
public interface IRequestManager { Task<string> SendRequestAsync(string url); }
```
Подключение сервиса через DI и вызов функционального метода
```cs
///...
        // Получаем сервис из контейнера DI и используем его
        var requestManager = host.Services.GetRequiredService<IRequestManager>();
        string response = await requestManager.SendRequestAsync("Sampleurl");

        Console.WriteLine(response);
    }

    static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureServices((hostContext, services) =>
            {
                // Регистрируем HttpClient
                services.AddHttpClient();

                // Регистрируем RequestManager как transient
                services.AddTransient<IRequestManager, RequestManagerNew>();
            });
}

```
Недавно пришлось реализовывать дублирование менеджера, так произошло из за того, что сервис получения настроек(конфиги, урлы, наименования аттрибутов json) был переписан(с сохранением старой версии)для новых интеграций написанных на новом движке, благо чтобы не забы\ить пометил все что нужно obsolete.
```cs
// Реализация 1
public class RequestManagerNew : IRequestManager
{
    private readonly HttpClient _httpClient;

    public RequestManager(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }

    public async Task<string> SendRequestAsync(string url)
    {
        HttpResponseMessage response = await _httpClient.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}
// Реализация 2
public class RequestManagerOld : IRequestManager
{
    private readonly HttpClient _httpClient;
    private readonly Options _options;
    

    public RequestManager(HttpClient httpClient,ContextSettings _settings)
    {
        _httpClient = httpClient;
        _options = _settings.GetCurrentOptionNameSpace();
    }

    public async Task<string> SendRequestAsync(string url)
    {
	    url.Headers.add(_options.GetAuthHeader());
        HttpResponseMessage response = await _httpClient.GetAsync(url);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadAsStringAsync();
    }
}

```

# Итог

Шаблонный вариант сверху есть отражение Manager а, отвечающего за запросы во внутреннюю БД, SettingAccessor - реализовавшийся с новой логикой для нового движка и EntityClient, сервис отвечающий за API с сущностями.

Я думал что реализация(дублирование) менеджера спасет меня от ада связанности, но не думал что в новом коде будет все равно вызываться старый вариант, а все из за того что глобально у меня жестко генерировался AddScoped, который перебивал локальный вызов, решением оказалось TryAddScoped, но чувствуется что я был очень близок к DI Hellу
