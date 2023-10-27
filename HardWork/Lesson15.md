# 1
Точка входа для wpf приложения - AppStartup
в этом методе пайплайн, который проверяет какие аргументы были переданы в это приложение, и в зависимости от аргумента/ов - специфичное действие(открытие документа, создание документа по определенному пути определенного формата...)
## было
```cs
async void AppStartup(object sender, StartupEventArgs e)
        {
        ///...
				string workingDirectory = GetExistDirectoryFromStartUpArguments(e);

                string executablePath;

                // _______________________________________________________________________________________
                // Случаи когда в аргументах триггер слова.
                if (CreateDOCXFromStartUpArguments(e) && await CreateDocxScenario(workingDirectory, coreConfig))
                {
                    Application.Current.Shutdown();
                    return;
                }
                ///...
```
Проверка от дурака, на null/empty значения
```cs
/// <summary>
        /// Сценарий для создания DOCX.
        /// </summary>
        /// <param name="executablePath"></param>
        /// <param name="filePath"></param>
        private async Task<bool> CreateDocxScenario(string workingDirectory, Config coreConfig)
        {
            if (String.IsNullOrEmpty(workingDirectory))
            {
                //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(App)}. ##### workingDirectory is null or empty.");
                return false;
            }
            ///...
```
___
## стало

```cs
/// <summary>
        /// Проверяет, есть ли в аргументах ключ слово - создать шаблонный документ DOCX и путь к папке в которой он создасться.
        /// </summary>
        /// <param name="e">Событие при открытии прокси.</param>
        /// <returns>Путь к документу.</returns>
        public static bool IsCreateDocumentScenary(StartupEventArgs e,out string workingDirectory)
        {
            bool hasCreateKeyInArgument = false;
            workingDirectory = String.Empty;
            for (int i = 0; i != e.Args.Length; i++)
            {
                if (string.Equals(e.Args[i], CREATE_DOCX_COMMAND_PROXY))
                {
                    //Logger.Trace($"Create DOCX From Start Up Arguments is exist.");
                    hasCreateKeyInArgument = true;
                    continue;
                }
                try
                {
                    string dirName = Path.GetDirectoryName(e.Args[i]);

                    if (Directory.Exists(dirName))
                    {
                        //Logger.Trace($"ExistDirectoryFromStartUpArguments: {e.Args[i]}.");
                        workingDirectory = dirName;
                    }
                }
                catch (Exception ex)
                {
                    //Logger.Error($"Error: {ex}.");
                    continue;
                }
                
            }
            return hasCreateKeyInArgument && !String.IsNullOrEmpty(workingDirectory);
        }
```

```cs
async void AppStartup(object sender, StartupEventArgs e)
        {
        ///...
        
 string workingDirectory;

                // _______________________________________________________________________________________
                // Случаи когда в аргументах триггер слова.
                if (IsCreateDocumentScenary(e,out workingDirectory) && await CreateDocxScenario(workingDirectory, coreConfig))
                {
                    Application.Current.Shutdown();
                    return;
                }
            ///...
```
переименован метод CreateDOCXFromStartUpArguments на IsCreateDocumentScenary.
Теперь этот метод не просто смотрит есть ли в аргументах 1 ключ триггер, а несколько, поскольку комбинация аргументов задает одну логику. Логика метода GetExistDirectoryFromStartUpArguments была обьединена с методом выше.

Переменная executablePath была перенесена дальше вниз, ближе к первому методу который взаимодействует с ней.

Таким образом мы избавились от проверки на пустоту исключив вообще ситуацию что для создания документа не передастся путь для его будущего местоположения.

# 2

Происходило так что мы получали сообщения от шаблонных, временных документов, или вообще от самого исполняемого файла, потому приходилось прописывать ряд условий в пайплайн обработки сигнала.
Решением же стало создание отдельного класса FileNameChecker, который включает в себя ряд методов, которые будут отсеивать сигналы ряда документов/исполняемых файлов.
```cs

private const string TEMPORARY_SYMBOLS = "~$";
        private const string TEMPORARY_START_FILE_NAME = "~";
        private const string TEMPORARY_END_FILE_NAME = ".tmp";
        // Процессов не так много, добавлять по мере необходимости.
        private const string WORD_EXECUTABLE_NAME = "WINWORD.exe";
        // Уникальных путей так же не очень много.
        private const string MICROSOFT_TEMPLATE_PATH = @"AppData\Roaming\Microsoft\Templates\";
        private const string MICROSOFT_OFFICE_RECENT_PATH = @"AppData\Roaming\Microsoft\Office\Recent\";
    /// <summary>
        /// Является ли файл временным.
        /// </summary>
        /// <param name="fileName">Имя файла.</param>
        /// <returns>True - если является.</returns>
        public static bool IsFileTemporary(string fileName)
        {
            if (fileName.Contains(TEMPORARY_START_FILE_NAME) ||
                fileName.Contains(TEMPORARY_END_FILE_NAME) ||
                fileName.Contains(TEMPORARY_SYMBOLS))
            {
                //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileNameChecker)}. File is temporary.");
                return true;
            }

            return false;
        }
```

# 3
Документ который пользователь хочет открыть может быть заблокирован по нескольким причинам:
1. Занят другим потоком, который читает метадату документа
2. Занят другим сторонним процессом
3. Занят другим экземпляром приложения
Для этого добавлены попытки спустя N количество мс 
```cs
/// <summary>
        /// Открытие документа.
        /// </summary>
        /// <returns></returns>
        private bool OpenDocument()
        {
            int countOfTryies = 0;
            while (true)
            {
                try
                {
                    if (countOfTryies >= 100)
                    {
                        //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(MarkManager)}.{nameof(OpenDocument)}. ##### countOfTryies > 100 SOMETHING WAS WRONG");
                        return false;
                    }
                    // попытка открытия документа, в случае неуспеха 
                    countOfTryies++;
                    Thread.Sleep(200);
                    continue;
                    ///...
```
Если для первых двух ситуаций добавление попыток достучаться к документу уместна, то для 3 ситуации нет, поскольку это ошибочное состояние приложения, которое не должно быть в принципе, не должно быть такого, чтобы несколько экземпляров приложения пытались открыть документ, однако такое может случиться из за ошибочного поведения пользователя(нажал на документ больше 2 раз ЛКМ).
Как решить ситуацию когда нужно открывать много разных документов и при этом блокировать повторное открытие открывшегося документа?
Мьютексы по пути файла.

```cs
bool createdNew;
                Mutex mutex = null;
                mutex = new Mutex(true, filePath, out createdNew);

                if (!createdNew)
                {
                    mutex.Close();
                    //Logger.Trace($"##### Другой процесс уже работает с этим файлом.");
                    Application.Current.Shutdown();
                    return;
                }
                ///...
```

# 4
Заменим состояние расширения с типа string на перечисление, ограничив варианты.
Было
```cs
	/// <summary>
        /// Получить ссылку на исполняемый файл в зависимости от расширения документа.
        /// </summary>
        /// <param name="fileExtension">Расширение документа.</param>
        /// <returns></returns>
        public static string GetExecutablePath(string fileExtension)
        {
            if (String.Equals(fileExtension, ".docx")|| String.Equals(fileExtension, ".odt")) return GetWordExe(
                Config<Config>.Config.DOCX_WORD_NameInRegistry,
                Config<Config>.Config.DOCX_WORD_ExecutableFileName);
            if (String.Equals(fileExtension, ".pdf")) return GetAdobeExe(
                 Config<Config>.Config.PDF_ADOBE_PathInstaller,
                 Config<Config>.Config.PDF_ADOBE_ExecutableFileName
                 );
            return String.Empty;
        }
```
Стало
```cs
public enum eDocumentExtensions
    {
        DOCX = 1,
        ODT = 2,
        PDF = 3,
        XLSX = 4,
        PPTX = 5,
        XLS = 6,
        DOC = 7,
        PPT = 8,
        XLSM = 9,
        DOCM = 10,
        ODS = 11,
        ODP = 12
    }
```

```cs
/// <summary>
        /// Определить расширение по имени файла.
        /// </summary>
        /// <param name="fileName">Имя файла.</param>
        /// <returns>Расширение файла.</returns>
        public static eDocumentExtensions GetDocExtention(string fileName)
        {
            int idxExt = fileName.LastIndexOf('.');
            string fileExt = fileName.Substring(idxExt + 1)
                                     .ToUpper();

            foreach (eDocumentExtensions ext in Enum.GetValues(typeof(eDocumentExtensions)))
            {
                if (String.Equals(fileExt, ext.ToString()))
                {
                    //Logger.Trace($"File extention - {ext}.");
                    return ext;
                }
            }

            throw new Exception($"Error get extention in enum {nameof(eDocumentExtensions)} for filename:{fileName} and it extention:{fileExt}.");
        }
```
Проверка на null Для fileName отсутствует из за того что мы отсеяли это состояние в инициализации приложения.

```cs
		/// <summary>
        /// Получить ссылку на исполняемый файл в зависимости от расширения документа.
        /// </summary>
        /// <param name="fileExtension">Расширение документа.</param>
        /// <returns></returns>
        public static string GetExecutablePath(eDocumentExtensions fileExtension)
        {
            switch (fileExtension)
            {
                case eDocumentExtensions.DOCX:
                case eDocumentExtensions.ODT:
                    return GetWordExe(
                Config<Config>.Config.DOCX_WORD_NameInRegistry,
                Config<Config>.Config.DOCX_WORD_ExecutableFileName);
                case eDocumentExtensions.PDF:
                    return GetAdobeExe(
                 Config<Config>.Config.PDF_ADOBE_PathInstaller,
                 Config<Config>.Config.PDF_ADOBE_ExecutableFileName
                 );
                default:
                    return String.Empty;
            }
        }
```