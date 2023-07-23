
## 1
По началу не задавался вопросом во время создания мониторинга процессов, а почему он вообще должен мониторить? Почему при смерти процесса мы должны стирать кеш по PID'у
```cs
    /// <summary>
    /// Класс мониторящий события создания\удаления процессов.
    /// </summary>
public class ProcessMonitor{

//  ...

		/// <summary>
        /// Метод, при получении уведомления об удалении процесса прокси, отправляет событие закрытия документа,
        /// поскольку по логическому дизайну у нас связь: один документ - один процесс.
        /// </summary>
        /// @attention Выход из "бесконечного цикла" осуществляется по флагу cancellationToken.
        private void StartProxyWatcherWin()
        {
            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(StartProxyWatcherWin)}. ->");
            string processName = "Proxy.exe";
            string query = $"SELECT * FROM __InstanceDeletionEvent WITHIN 1 " +
                $"WHERE TargetInstance ISA 'Win32_Process' " +
                $"AND (TargetInstance.Name = '{processName}')";
            try
            {
                using (ManagementEventWatcher watcher = new ManagementEventWatcher(query))
                {
                    watcher.EventArrived += (sender, e) =>
                    {
                        ManagementBaseObject targetInstance = (ManagementBaseObject)e.NewEvent.Properties["TargetInstance"].Value;
                        string processName = targetInstance["Name"].ToString();
                        int processId = Convert.ToInt32(targetInstance["ProcessId"]);
                        //Logger.Trace($"##### Proxy with PID: {processId} is dead.");
                        if (_workingProxyCache.workingProxyCache.ContainsKey(processId))
                        {
                            //Logger.Trace($"##### Sending event document {_workingProxyCache.workingProxyCache[processId]} is closed.");
                            _notificationProcessor.AddSendEventMessage(new ClientEventMessage(_workingProxyCache.workingProxyCache[processId], eEventType.CloseDocEvent));
                            //Logger.Trace($"##### Removing {processId} from ProxyPIDCache");
                            _workingProxyCache.workingProxyCache.Remove(processId);
                        }
                    };
                    watcher.Start();

                    while (!_cancellationTokenSrc.IsCancellationRequested)
                    {
                        // Необходимо добавить некоторую паузу в выполнении потока, иначе ЦПУ сильно нагружается
                        Thread.Sleep(500);
                    }
                    watcher.Stop();
                }

            }
            catch (Exception e)
            {
                //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(ProcessMonitor)}.{nameof(StartProxyWatcherWin)}. Error: {e}");
            }
        }

```
После комментирования метода StartProxyWatcherWin становится виднее и более понятно на глобальном уровне ПОЧЕМУ мы ловим удаление процесса и в последствии чистим кеш.

## 2
Комментирование глобального характера напрашивалось само по себе, логично что нужно объяснить почему мы создаем класс и пытаемся достучаться до документа без каких либо дальнейших действий, в других проектах это был бы рудимент, но тут  важнейшая часть кода, поскольку проверка метаданных происходит в другом проекте.
```cs
        /// <summary>
        /// Стартует процесс открытия документа.
        /// </summary>
        /// <param name="executablePath">Путь к исполняемому файлу.</param>
        /// <param name="fileExtension">Расширение файла.</param>
        /// <param name="filePath">Путь к документу.</param>
        public static void OpenDocument(string executablePath, string fileExtension, string filePath)
        {
            //Logger.Trace($"Trying read file");
            try
            {
                // Посылаем сигнал драйверу, если драйвер(агент) откажет в открытии, то
                // Выскочет исключение access
                // Для того чтобы проверить доступ к файлу по типу метки, мы создаем обьект который будет стучаться
                // к файлу. Драйвер получит попытку вызова(где PID - наш прокси), прочитает тип метки, 
                // доступ текущего пользователя и, в случае успеха, предоставит доступ, 
                // в случае неудачи выскочет ошибка об отстутствии доступа
                using (StreamReader reader = new StreamReader(filePath))
                {

                }
```
Уверен что без этого комментария другой человек подумал бы что это какой то рудимент, и с чистой совестью удалил этот участок кода, однако в дальнейшем он потратит минимум день на то чтобы понять, а почему все сломалось. И чья это вина если не моя.
# 3 
В комментарии указал, почему этот метод ДОЛЖЕН вызываться а так же ЧТО ИМЕННО должен делать, поскольку без очистки кеша ворда выскакивает аномальное поведение.
```cs
/// <summary>
        /// Из за вызова этой функции почему то документ открывается в ридонли, может нужно пропускать первый item?
        /// Word хранит в реестре документы с которыми он работал ранее, поскольку PID прокси связан только с одним
        /// документом то может выскакивать очень много уведомлений об отказе в доступе к документу (Экземпляр ворда
        /// стучится к открывавшимся ранее документам 2 3 4, хотя в кеше его пид связан с документом 1).
        /// </summary>
        public static void ClearWord16OpenedFilesCache()
        {
            const string registryPath = @"SOFTWARE\Microsoft\Office\16.0\Word\File MRU";
            const string valuePrefix = "Item ";

            RegistryKey key = Registry.CurrentUser.OpenSubKey(registryPath, true);

            // Проверяем, что ключ существует
            if (key != null)
            {
                // Получаем список всех значений внутри ключа
                string[] valueNames = key.GetValueNames();

                foreach (string valueName in valueNames)
                {
                    // Проверяем префикс значения
                    if (valueName.StartsWith(valuePrefix))
                    {
                        //Logger.Trace($"Deleting {valueName}.");
                        // Удаляем значение
                        key.DeleteValue(valueName);
                    }
                }

                key.Close();
                return;
            }
            //Logger.Error($"key {registryPath} == null !!!");
        }
```
В последнем выводе хотел бы наверно дополнить что немаловажно рассказать и о негативном сценарии, при пропуске/изменении этого метода, что может произойти если проигнорировать/изменить поведение функции именно в глобальной работе программы.

Комментарии писать нужно. не КАК а ПОЧЕМУ.