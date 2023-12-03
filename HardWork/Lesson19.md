# 2

# 2.1
Для отслеживания печати существует отдельный класс printMonitor, во время проверки мы ищем имя в списке являющийся полем этого класса. Добавление происходит только во время обработки сигнала драйвера, и только тогда когда у пути к этому файлу есть метадата и только когда ответ не default а allow. Путаницы из за этого было много, приходилось искать ссылки кто куда кому отправляет на 2-3 шага.

```cs
    /// <summary>
    /// Монитор печати.
    /// </summary>
    /// Используется службой для контроля всех файлов, отправленных пользователем на печать.
    internal class PrintMonitor
    {
        /// <summary>
        /// Список имён файлов.
        /// </summary>
        private List<string> _cacheFileNames = new List<string>();
	///...
	if (CheckPath(documentName)){
		printJob.InvokeMethod("Resume", null);
        _printFileName = _cacheFileNames.Find(c => documentName.Contains(Path.GetFileNameWithoutExtension(c)));
	///...
}
```


```cs
        private bool CheckPath(string spoolerDocName)
        {
            string filePath = _cacheFileNames.Find(c => spoolerDocName.Contains(Path.GetFileNameWithoutExtension(c)));
            ///...
```

```cs
/// <summary>
        /// Обработчик события клиента драйвера о получении нового файла.
        /// </summary>
        /// <param name="sender">Отправитель события является клиента драйвера.</param>
        /// <param name="args">Параметры события содержат строку полного имени нового файла, к которому драйвер зафиксировал обращение пользователя.</param>
        private void NotificationProcessor_FullPathReceived(object sender, FileNotifyEventArgs args)
        {
            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(PrintMonitor)}.{nameof(NotificationProcessor_FullPathReceived)}. ->");

            if (!String.IsNullOrWhiteSpace(args.FileFullPath))
            {
                _resetEvent.Reset();
                if (!_cacheFileNames.Contains(args.FileFullPath))
                {
                    _cacheFileNames.Add(args.FileFullPath);
                }
                _resetEvent.Set();

                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(PrintMonitor)}.{nameof(NotificationProcessor_FullPathReceived)}. New File full path:{args.FileFullPath}");
            }

            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(PrintMonitor)}.{nameof(NotificationProcessor_FullPathReceived)}. <-");
        }
```
___

```cs
                case eProccessingResult.Allow:
                    {
                    ///...
                    // Сохранить для печати
                        if (result.Metadata != null)
                        {
                            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}] Save file info for print monitoring");
                            RiseFullPathReceived(notifyMsg.FullFilePath);
                        }
```
Довольно хрупкая и ненадежная конструкция, в качестве исправления был выделен отдельный потокобезопасный кеш открытых документов, присылающихся через pipeline.
# 2.2
в легаси перегрузке метода добавления в кеш, была добавлена булева которая жестко назначала время жизни 1 день.
```cs

/// <summary>
    /// Кеш обработанных файлов.
    /// </summary>
    public class FileDataCache : IFileDataCache
    {
	    public void Add(string fileName, MetadataDto metadata, bool isPrint = false)
	        {
        
	            if (isPrint)
	            {
                elem = new FileDataCacheElement
	                {
                    Ttl = DateTime.UtcNow.AddDays(1),
                    FileName = fileName,
                    Metadata = metadata
	                };
	            }
```
в качестве фикса была реализована логика проверки лока файла при обновлении/вытаскивании элемента из кеша.

```cs
private bool IsExpired(string key, T value)
        {
            if (DateTime.UtcNow < value.Ttl.AddSeconds(-30)) return false;

            Process[] winProcesses = Process.GetProcesses();

            System.Collections.Generic.List<Process> processList = FileUtil.WhoIsLocking(value.FullPath);

            if (processList.FirstOrDefault(c => TrustedApplications.Apps.Contains(c.ProcessName)) == null ||
                winProcesses.FirstOrDefault(c => c.Id == value.PID) == null)
            {
                return this.TryRemove(key, out value);
            }
            value.UpdateTTLManually(DateTime.UtcNow.AddSeconds(_ttlSec));
            
            return false;
        }
```
# 3 

## 3.1
чтение таблички не по индексу а по имени
```cs
public static bool StartParseRequests(string excelFilePath, out List<Request> requests)
        {
            requests = new List<Request>();
            int currentRow = 1;
            try
            {
                using (SpreadsheetDocument spreadsheetDocument = SpreadsheetDocument.Open(excelFilePath, false))
                {
                    WorkbookPart workbookPart = spreadsheetDocument.WorkbookPart;
                    var sheets = workbookPart.Workbook.Descendants<Sheet>();
                    var sheet = sheets.ToList()[2];
```

для метода StartParseClients аналогично
```cs
var sheet = sheets.ToList()[1];
```
итд...
Конечно же решением будет поиск не по индексу а по имени таблицы в excel. Таким образом мы избежим глупой ошибки если порядок sheet будет изменен при парсинге xlsx файла.

# 3.2


Допустим по спецификации мы из очереди сообщений получаем чек покупки продуктового магазина. Параметров в этом сообщении довольно много и есть степень важности некоторых полей(ценник товара, сумма товаров к оплате, скидка, купоны...)

Однако если при реализации мы забудем(плохо реализуем валидацию второстепенных полей таких как адрес магазина, инн...) то получается что в базу данных для аналитиков будут добавляться не истинные данные, а от них может зависеть дальнейшая разработка будущих фич.
# 4

# 4.1 Кеш файлов.
На данный момент из за отсутствия спецификации реализация все растет и растет, а именно добавляется срок сгорания + условия при которых элемент выйдет из жизни(занят ли текущий файл каким то процессом или нет.). Так же никто не отменяет будущее добавление веб хука от сервера(изменение безопасником правил доступа для файла.) Ну и конечно же сам кеш должен быть потокобезопасным, не просто add,tryget,update,remove, а обернутым в writelock readlock.
# 4.2 Чтение/Запись файла
Интерфейс для работы с файлом должен так же учитывать тот факт что файл может не успеть освободиться от закрывающегося процесса, нужно проверять кто занял файл и сколько попыток постучаться к этому файлу с какой переодичностью.
# 4.3 Возврат значения даже при exception
Конечно же мы можем чтото не учесть и реализация может выдать exception, но и после того как мы это исключение отловили в реализации нам все равно нужно чтото вернуть, потому для случаев 4.1 и 4.2 нужно создать отдельный тип который будет повествовать нам о результате операции (success,failed,timeout), в общем статус код.