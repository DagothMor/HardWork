# 1 Изменение отправки сигнала о документе между проектами
Существует *Учебная* утилита работающая как FileMon
https://learn.microsoft.com/ru-ru/sysinternals/downloads/procmon
Есть специальный агент который ловит события от нашей учебной утилиты, делает анализ сигналов и после отправляет ответ файловой системе - разрешить доступ или же нет.
Однако как игнорировать(обрабатывать если сигналы типа сохранения/создания...) сигналы которые будут идти от рабочих офисных приложений которые будут создавать тысячи сигналов к открытому документу?
## До
В приложении, которое анализирует доступ и решает с помощью какого офисного пакета открыть документ, создается объект байтового потока к открываемому документу.
```cs
try
                    {
                        // Посылаем сигнал драйверу, если драйвер(агент) откажет в открытии, то
                        // Выскочет исключение access
                        //using (var _ = new StreamReader(filePath)) { }
```
таким образом наша утилита улавливает это и передает агенту, он же, понимая по PID что это наше приложение, считывает метаданные документа и политику SID,сравнивает, если права у пользователя на взаимодействие документа есть - отправляем ответ файловой системе о разрешении, и наоборот. В конце в любом случае добавляем в кеш путь к документу - его метаданные.

## После
Сложности возникли в связи с созданием документа, если промониторить по ProcMon можно посмотреть ряд сигналов от файловой системы, как происходит создание docx на рабочий стол(по большей части идет не создание, а копирование из папки с шаблонными документами AppData/Roaming/Microsoft/Templates/).
Вот тут и начинается гонка/ бег в перед паровоза. То сигналы приходят когда файл занят процессом проводника, то приходят на еще не существующий файл, якобы наш новый по нашему пути.
Так же как нам понять что сигнал относится к тому файлу который на данный момент открыт? Как понять что он закрылся? И именно рабочими офисными приложениями?
Решением стало работой с новым и клонирующим файлом на уровне приложения а не агента. Но в таком случае придется игнорировать вызовы на уровне агента, а он должен знать метадату при будущем открытии файла. Здесь же поможет межпроцессорная коммуникация крутящаяся в отдельных задачах.
https://learn.microsoft.com/ru-ru/dotnet/api/system.io.pipes.namedpipeserverstream?view=net-7.0
```cs
private async Task FileMetadataCacheMonitor()
        {
            while (!_cancellationTokenSrc.IsCancellationRequested)
            {
                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(FileMetadataCacheMonitor)}. ->");
                await using var pipeClient = new NamedPipeClientStream(".", "FileMetadataPipe", PipeDirection.In);

                // Connect to the pipe or wait until the pipe is available.
                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(FileMetadataCacheMonitor)}. Attempting to connect to pipe...");
                await pipeClient.ConnectAsync();

                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(FileMetadataCacheMonitor)}. Connected to pipe.");
                using var sr = new StreamReader(pipeClient);

                var filePath = await sr.ReadLineAsync();

                if (string.IsNullOrEmpty(filePath))
                {
                ///...
                // Взаимодействуем с кешем
                ///...

private async Task OpenFileByProxyMonitor()
        {
            while (!_cancellationTokenSrc.IsCancellationRequested)
            {
                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(OpenFileByProxyMonitor)}. ->");
                await using var pipeClient = new NamedPipeClientStream(".", "Filepath_PidProxy_OpenPipe", PipeDirection.In);
                ///...
                // Взаимодействуем с кешем
                ///...

private async Task CloseFileByProxyMonitor()
        {
            while (!_cancellationTokenSrc.IsCancellationRequested)
            {
                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(CloseFileByProxyMonitor)}. ->");
                await using var pipeClient = new NamedPipeClientStream(".", "Filepath_PidProxy_ClosePipe", PipeDirection.In);
                ///...
                // Взаимодействуем с кешем
                ///...
                


```

Вызов
```cs
case PlatformID.WinCE:
                    {
                        _fileMetadataCacheWatch = Task.Factory.StartNew(FileMetadataCacheMonitor);
                        _openFileByProxyWatch = Task.Factory.StartNew(OpenFileByProxyMonitor);
                        _closeFileByProxyWatch = Task.Factory.StartNew(CloseFileByProxyMonitor);
                    }
                    break;
```

Возникает другая ситуация - конкуренция за кеш.
Решение - сделать кеш потокобезопасным 
https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2?view=net-7.0

Вывод в очередной раз интересен. Перенесли ряд логики из одной программы в другую, и по итогу приходится менять структуру данных и pipeline 

# 2 Дидактические карточки везде и всегда.
В прошлом 16 ответе мы добавляли Dependency injection. Представим что нам потребуется проверять дидактические карточки целый день в независимости от того за чем мы сидим, телефон(телеграм бот), свой компьютер(локальное приложение), офисный(допустим, припоминаем через браузер, развернули сайт на своем компьютере.)
И если работа через телеграм бот или браузер понятна(сплошные апи запросы на сервер), то работа с локальным компьютером поинтереснее, поскольку мы можем пометить галкой что на нашем локальном компьютере будет мастер бд, и работать мы будем с ней как на локальном, так и через апи.
Как обновлять мастер бд, учитывая что мы можем работать как с телефона так и с локального одновременно/часто переключаясь.
Как вариант - работа с бд только через api, как принимать запросы по сети так и через межпроцессорные протоколы.
Однако нам нужно своевременное обновление(добавили вопрос-ответ через телеграм, получили файлик на локальном компьютере.)
Решением будет создание вебхуков.
Таким образом мы разделили работу с бд отдельным сервисом,который так же сможет присылать своевременно уведомления браузеру/компьютеру
