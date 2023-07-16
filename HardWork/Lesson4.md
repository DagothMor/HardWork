Подумал касаемо того что из себя программа вообще должна представлять, начал с расплывчатого ТЗ.

Прокси должен являться управляемой оболочкой для пользователя и определенным флагом для драйвера. Пользователь должен работать только с теми расширениями и программами которыми разрешит администратор/ корпоративная политика. Драйвер благодаря прокси разделяет манипуляции с файлами и явную от пользователя попытку открыть/закрыть документ.

Попробую переписать логику запуска прокси

Логический дизайн прокси:
Прокси запускается получая значения в аргументах.
В зависимости от полученных аргументов(ключ слов) происходят разные сценарии например:

"Create docx","create pdf "... - Создать документ с определенным расширением и открыть его.
"ПУТЬ/ИМЯ_ФАЙЛА.РАСШИРЕНИЕ" - Открыть документ по умолчанию.
"ПУТЬ/ИМЯ_ФАЙЛА.РАСШИРЕНИЕ", "ИМЯ_ПРИЛОЖЕНИЯ" - Открыть документ с определенным приложением.

Несложно понять что сценариев 3, но они могут увеличиться, как правильно в таком случае поступитacь? Не будем фильтровать по количеству аргументов, скорее по наличию файла с которым нужно будет работать.

```
Начало программы.

Если аргументов 0 -> выводим ошибку

Если в аргументах нет существующего файла -> Пытаемся проинициализировать аргументы в списке ключ-слов
и после завершаем программу(ветка без файла).

Если расширение существующего файла не поддерживается -> Выводим ошибку.

Пытаемся получить имя поддерживаемой исполняемой программы из аргументов. 

В случае неудачи, указываем исполняемую программу по умолчанию с помощью раннее созданного
класса programManager

Создаем экземпляр FileManager передавая расширение и программу.

Пытаемся открыть документ.
Ожидаем закрытия документа.

Конец программы.
```

Во время написания сценария стало понятно что пользователь может захотеть создать шаблонный документ в определенной папке, возможно ли это и отлаживается ли, пока неизвестно из за редактора реестра, однако расширять и изменять логику уже станет легче из за разделения сценариев.
посмотрим на текущую реализацию

___

# КОД ДО 
Итого стартовый метод работал таким образом:
```cs 
void AppStartup(object sender, StartupEventArgs e)
        {
            var proxyConfig = // инициализация корпоративного конфигурационного файла 1
            var coreConfig = // инициализация корпоративного конфигурационного файла 2

            string filePath = FullFilePathFromStartUpArguments(e);

            if (!FilePathIsExist(filePath))
            { 
            // Только для тестов, убрать при релизе. сделать список поддерживаемых расширений и пройтись по нему если exe с пустыми аргументами.
                CheckRegistry(".docx"); CheckRegistry(".pdf"); 
                return;
            }

            string fileExtension = Path.GetExtension(filePath).ToLower();
            if (!FileExtensionIsSupported(fileExtension)) return;

            // Только для тестов, убрать при релизе.
            CheckRegistry(fileExtension);

            string executablePath;
            if (ExecutablePathIsExist(fileExtension, out executablePath)) return;

            //bool isAuthorized = false;

            //using (MarkManager mc = new MarkManager(coreConfig.ClientUrl, filePath))
            //{
            //    isAuthorized = mc.AuthorizationUserByDocument();
            //}
            try
            {
                OpenDocument(executablePath, fileExtension, filePath);
            }
            catch (Exception ex)
            {
                //Logger.Trace($"Exception:{ex}");
                MessageBox.Show("ACCESS DENIED.", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
                //throw;
            }
            // Аномальное плавающее поведение, экземпляр открывается то в ридонли то стандартно.
            //RegistryHelper.ClearWord16OpenedFilesCache();
            Application.Current.Shutdown();

        }
```

Как видим беспорядок, в целях лжеэкономии времени проверяли работу чисто с word, но никак не с другими приложениями. Хотя недавно в рамках исследования пробовал создавать файл через прокси, создавая ярлык и перекидывая в аргумент ключ слово "Create", переписывать логику и отлаживать пару часов было неприятно, страшно представить что было бы будь это более глобальным и масштабным приложением.

для сценария с файлом создадим класс и интерфейс, в нем будет метод open document.
```cs
public interface IFileManager
    {
        /// <summary>
        /// Открыть документ.
        /// </summary>
        void OpenDocument();
    }
```

```cs
    /// <summary>
    /// Менеджер для документа.
    /// </summary>
    public class FileManager : IFileManager
    {
        private string _executablePath;
        private string _filePath;
        
        public FileManager(string filePath,string executablePath)
        {
            _filePath = filePath;
            _executablePath = executablePath;
        }
        public void OpenDocument()
        {
            //Logger.Trace($"Trying read file");
            try
            {
                // Посылаем сигнал драйверу, если драйвер(агент) откажет в открытии, то
                // Выскочет исключение access
                using (StreamReader reader = new StreamReader(_filePath))
                {

                }
                //Logger.Trace($"executablePath: {_executablePath}, fileExtension:{_fileExtension}, {_filePath}");
                Process myProcess = new Process();
                myProcess.StartInfo.FileName = _executablePath;
                myProcess.StartInfo.Arguments += "\"" + _filePath + "\"";
                myProcess.Start();
                myProcess.WaitForExit();
            }
            catch (Exception ex)
            {
                //Logger.Trace($"##### Exception {ex}");
                MessageBox.Show("ACCESS DENIED", "ERROR", MessageBoxButton.OK, MessageBoxImage.Error);
                return;
            }
        }
    }
```

# КОД ПОСЛЕ
```cs
        void AppStartup(object sender, StartupEventArgs e)
        {
            var proxyConfig = // инициализация корпоративного конфигурационного файла 1
            var coreConfig = // инициализация корпоративного конфигурационного файла 2

            // Отладка списка программ по умолчанию, в будущем реализовать конструктор который
            // принимает класс Config из proxyConfig и сам парсит в словарь.
            var programManager = new ProgramManager();
            FileManager fileManager;

            if (e.Args.Length == 0)
            {
                //Logger.Error($"Args.Length == 0");
                MessageBox.Show("ARGS not found.", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
                Application.Current.Shutdown();
            }

            string filePath = FullFilePathFromStartUpArguments(e);

            if (String.IsNullOrEmpty(filePath))
            {
                //TODO: здесь можно реализовывать различные действия если в аргументах
                // будут находиться слова-триггеры.
                Application.Current.Shutdown();
                return;
            }

            string fileExtension = Path.GetExtension(filePath).ToLower();

            if (!programManager.ExtensionIsSupported(fileExtension))
            {
                MessageBox.Show("Extension of this document is not supported.", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
                Application.Current.Shutdown();
                return;
            }
            
            var supportedProgram = programManager.GetSupportedProgramFromArgs(e.Args);

            if (String.IsNullOrEmpty(supportedProgram))
            {
                supportedProgram = programManager.GetDefaultProgram(fileExtension);
            }

            fileManager = new FileManager(filePath, supportedProgram);

            fileManager.OpenDocument();

            // Аномальное плавающее поведение, экземпляр открывается то в ридонли то стандартно.
            //RegistryHelper.ClearWord16OpenedFilesCache();
            Application.Current.Shutdown();

        }
```

Как видно из кода после, логика стала максимально приближенной к дизайну. Переписывание заняло много времени из за описания ТЗ и логической архитектуры, которая сподвигла на дальнейшее обдумывание ЧТО нужно сделать(какую смысловую нагрузку и для каких случаев должен взять на себя интерфейс менеджера файла.)
___

# Сценарий ключ слов
Во время исследования создания файла учитывая FileFilterSystem была реализована логика создания/пересоздания файла из шаблона через прокси

```
Если в аргументах нет существующего файла -> Пытаемся проинициализировать аргументы в списке ключ-слов
и после завершаем программу(ветка без файла).
Если есть ключ слово create docx ->
- Создаем новый процесс ProxyFileCreator
- Передаемв аргументы расширение по которому нужно создать документ
- ждем выполнения
- если в шаблонном пути нет шаблонного файла выводим ошибку и завершаем работу
- открываем шаблонный документ по умолчанию
```
# КОД ДО
```cs
bool createDOCXFromStartUpArguments = CreateDOCXFromStartUpArguments(e);
            if (createDOCXFromStartUpArguments)
            {
                Process myProcess = new Process();
                myProcess.StartInfo.FileName = PROXY_FILE_CREATOR_PATH;
                myProcess.StartInfo.Arguments += "docx";
                myProcess.Start();
                myProcess.WaitForExit();

                // Проверяем, удачно ли создался файл.
                if (!File.Exists(CREATED_DOCUMENT_PATH))
                {
                    //Logger.Error($"##### File is not exist!!! : {CREATED_DOCUMENT_PATH}");
                    Application.Current.Shutdown();
                }

                filePath = CREATED_DOCUMENT_PATH;
                if (ExecutablePathIsExist(DOCX_EXTENSION, out executablePath)) return;
                try
                {
                    OpenDocument(executablePath: executablePath, fileExtension: DOCX_EXTENSION, filePath: filePath);
                }
                catch (Exception ex)
                {
                    //Logger.Error($"Exception:{ex}");
                    MessageBox.Show("ACCESS DENIED.", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
                    throw ex;
                }
                // чистим кеш в word
                RegistryHelper.ClearWord16OpenedFilesCache();
                Application.Current.Shutdown();
                return;
            }

```

ProxyFileCreator отдельный проект который будет создаваться ради пере/создания файла в шаблонном стандартном пути из внутреннего хранилища шаблонных документов.
Вариант создания шаблонного файла через ProxyFileCreator обязателен поскольку его имя мы зарегистрируем в драйвере, он же в свою очередь будет игнорировать все действия которые приложение будет выполнять.
```
Если в аргументах есть docx -> 
- Если файл существует в шаблонном пути то удаляем его
- Копируем файл в шаблонный путь из внутреннего хранилища.
```
Реализация
```cs
static void Main(string[] args)
        {
            if (args.Contains("docx")) 
            {
                CreatingTemplateDocument();
            }
        }
        private static void CreatingTemplateDocument()
        {

            string sourceFile = TEMPLATE_DOCX_PATH;
            string destinationFileName = CREATED_DOCUMENT_PATH;
            try
            {
                // Проверяем, существует ли файл
                if (File.Exists(destinationFileName))
                {
                    // Если файл существует, удаляем его
                    File.Delete(destinationFileName);
                }
                File.Copy(sourceFile, destinationFileName);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
                throw ex;
            }
        }
```
Благодаря предыдущей разработке вместо метода создадим экземпляр ранее проработанного FileManager

# КОД ПОСЛЕ
```cs
//TODO: здесь можно реализовывать различные действия если в аргументах
                // будут находиться слова-триггеры.
                if (CreateDOCXFromStartUpArguments(e))
                {
                    Process myProcess = new Process();
                    myProcess.StartInfo.FileName = PROXY_FILE_CREATOR_PATH;
                    myProcess.StartInfo.Arguments += "docx";
                    myProcess.Start();
                    myProcess.WaitForExit();

                    // Проверяем, удачно ли создался файл.
                    if (!File.Exists(CREATED_DOCUMENT_PATH))
                    {
                        //Logger.Error($"##### File is not exist!!! : {CREATED_DOCUMENT_PATH}");
                        Application.Current.Shutdown();
                    }

                    filePath = CREATED_DOCUMENT_PATH;
                    supportedProgram = programManager.GetDefaultProgram(fileExtension);
                    fileManager = new FileManager(filePath, supportedProgram);

                    fileManager.OpenDocument();

                    // Аномальное плавающее поведение, экземпляр открывается то в ридонли то стандартно.
                    //RegistryHelper.ClearWord16OpenedFilesCache();
                    Application.Current.Shutdown();
                    return;
                }
                if (CreatePDFFromStartUpArguments(e))
                {
                    Application.Current.Shutdown();
                    return;
                }
                Application.Current.Shutdown();
                return;
```
Понял что лучше всего до начала кода писать логический дизайн, особенно когда задача абстрактная с большим количеством внешних факторов, поскольку мне стало легче добавлять новую логику(ключслова вместо открытия документа), и в будущем, ведь что если пользователь захочет открыть новый документ в определенной программе? По логическому дизайну легко понять что и где добавить. Однако работается конечно тяжелее, нужно больше концетрироваться и расписывать возможные негативные сценарии, и еще больше плевать в потолок чтобы голова дальше размышляла в расфокусе...