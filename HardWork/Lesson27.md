# 1 Открытие файла
## До
```cs
	/// <summary>
    /// Управление файлом.
    /// </summary>
    public class FileManager : IFileManager
    {
        bool _isReadOnly = false;

        private IGenericDocument _document;

        private string _filePath;

        private bool _needSaveToFile;

        /// <summary>
        /// Текущие метаданные в памяти.
        /// </summary>
        /// Не путать с метаданными в самом файле. Этот параметр изменяется при работе с объектом.
        private NDA _metadata;


        /// <summary>
        /// Текущие метаданные в памяти.
        /// </summary>
        public NDA Metadata
        {
            get
            {
                try
                {
                    if (_document == null)
                    {
                        _metadata = Read();
                        if (_metadata == null)
                        {
                            _needSaveToFile = false;
                            return null;
                        }
                        if (_needSaveToFile)
                        {
                            Write(_metadata);
                        }
                    }
                    return _metadata;
                }
                catch (Exception ex)
                {
                    return null;
                }
            }
            set
            {
                if (value == null)
                {
                    return;
                }

                if (_document == null)
                {
                    _metadata = Read();
                    if (_metadata == null)
                    {
                        throw new ArgumentException($"metadata is null.");
                    }
                }
                if (!_needSaveToFile)
                {
                    //Logger.Trace("File is open only for read.");
                    return;
                }
                Write(_metadata);
            }
        }
```

```cs
		/// <summary>
        /// Уничтожение объекта.
        /// </summary>
        public void Dispose()
        {
            if (_document != null)
            {
                if (_needSaveToFile)
                {
                    Save();
                }
                if (_document.IsOpen)
                {
                    _document.Close();
                }

                // После закрыти устанавливаем атрибут, если он был
                if (_isReadOnly)
                {
                    FileInfo fileInfo = new FileInfo(_filePath);
                    fileInfo.IsReadOnly = true;
                }
            }
        }
```

```cs
// ___
// методы, меняющие метаданные текущего документа...
// ___

        /// <summary>
        /// Прочитать метаданные из файла.
        /// </summary>
        /// <returns>Метаданные файла.</returns>
        private MetadataDto Read()
        {
        
```

Да, чтобы получить метаданные, в легаси вызывался метод read(), в котором первым делом проверялось поле `_filepath` на null..., после пытаемся открыть документ (это еще было без количества попыток открытий(попыток постучаться через N секунд, ибо процессы(word например, который открыл этот документ) могут не так быстро закрываться)).
в общем так ЧТЕНИЕ файла, превращается в ПРОВЕРКУ пути файла, его СУЩЕСТВОВАНИЕ, ПОПЫТКА открытия...

**"Отец, прости им, ибо они не ведают, что творят"**

## После
```cs
using System;
using System.IO;

// IFileState Interface
public interface IFileState
{
    void Open(FileContext context);
    void ChangeContents(FileContext context, string content);
    void Close(FileContext context);
}

// FileClosed Class
public class FileClosed : IFileState
{
    public void Open(FileContext context)
    {
        Console.WriteLine("File is opened.");
        context.State = new FileOpen(); // Change state to FileOpen
    }

    public void ChangeContents(FileContext context, string content)
    {
        Console.WriteLine("Cannot change contents. File is not open.");
    }

    public void Close(FileContext context)
    {
        Console.WriteLine("File is already closed.");
    }
}

// FileOpen Class
public class FileOpen : IFileState
{
    public void Open(FileContext context)
    {
        Console.WriteLine("File is already open.");
    }

    public void ChangeContents(FileContext context, string content)
    {
        // Example: Writing content to a file (you can include the file path in the context)
        Console.WriteLine("Changing file contents.");
        File.WriteAllText(context.FilePath, content);
    }

    public void Close(FileContext context)
    {
        Console.WriteLine("File is closed.");
        context.State = new FileClosed(); // Change state to FileClosed
    }
}

// FileContext Class
public class FileContext
{
    public IFileState State { get; set; }
    public string FilePath { get; set; }

    public FileContext(string filePath)
    {
        State = new FileClosed(); // Initial state
        FilePath = filePath;
    }

    public void Open()
    {
        State.Open(this);
    }

    public void ChangeContents(string content)
    {
        State.ChangeContents(this, content);
    }

    public void Close()
    {
        State.Close(this);
    }
}

// Usage
class Program
{
    static void Main(string[] args)
    {
        FileContext fileContext = new FileContext("example.txt");
        fileContext.Open(); // Opens the file
        fileContext.ChangeContents("Hello, World!"); // Changes the contents
        fileContext.Close(); // Closes the file
    }
}
```
# 2 IRPC пайпы через FSM
У пайп конечно же тоже есть состояния, попытка соединения, ожидание чтения, обработка, ожидание следующего сообщения.
Реализация До не показывает явно эти состояния, они конечно есть, но на абстрактном уровне этого кода.
## До
```cs
private async static Task<bool> SendPIDofClosedCurrentProcess(long PID)
        {
            try
            {
                var pipeServer = new NamedPipeServerStream("ClosedCurrentProcess_PID_ClosePipe", PipeDirection.Out);
                await pipeServer.WaitForConnectionAsync();
                var sw = new StreamWriter(pipeServer);
                sw.AutoFlush = true;
                sw.WriteLine(PID);
            }
            catch (IOException e)
            {
            }
            finally
            {
            }
            return true;
        }
```

```cs
while (!_cancellationTokenSrc.IsCancellationRequested)
            {
                await using var pipeClient = new NamedPipeClientStream(".", "ClosedCurrentProcess_PID_ClosePipe", PipeDirection.In);
                await pipeClient.ConnectAsync();
                
                using var sr = new StreamReader(pipeClient);

                var PIDString = await sr.ReadLineAsync();

                if (string.IsNullOrEmpty(PID))
                {
                    await Task.Delay(1000);
                    continue;
                }
                long PID;
                if (!Int64.TryParse(PID, out PID))
                {
                    await Task.Delay(1000);
                    continue;
                }
                ///
                // NDA
                ///

                await Task.Delay(1000);
            }
```

# После
```cs
public class PipeClientStateMachine
{
    private enum State
    {
        Connecting,
        Reading,
        Processing,
        Waiting
    }

    private State _currentState = State.Connecting;
    private CancellationToken _cancellationToken;

    public PipeClientStateMachine(CancellationToken cancellationToken)
    {
        _cancellationToken = cancellationToken;
    }

    public async Task RunAsync()
    {
        while (!_cancellationToken.IsCancellationRequested)
        {
            switch (_currentState)
            {
                case State.Connecting:
                    await ConnectAsync();
                    break;
                case State.Reading:
                    await ReadAsync();
                    break;
                case State.Processing:
                    Process();
                    break;
                case State.Waiting:
                    await WaitAsync();
                    break;
            }
        }
    }

    private async Task ConnectAsync()
    {
        await using var pipeClient = new NamedPipeClientStream(".", "ClosedCurrentProcess_PID_ClosePipe", PipeDirection.In);
        await pipeClient.ConnectAsync(_cancellationToken);
        _currentState = State.Reading;
    }

    private async Task ReadAsync()
    {
        await using var pipeClient = new NamedPipeClientStream(".", "ClosedCurrentProcess_PID_ClosePipe", PipeDirection.In);
        using var sr = new StreamReader(pipeClient);
        var PID = await sr.ReadLineAsync();

        if (!string.IsNullOrEmpty(PID))
        {
            _currentState = State.Processing;
        }
        else
        {
            _currentState = State.Waiting;
        }
    }

    private void Process()
    {
        _currentState = State.Waiting;
    }

    private async Task WaitAsync()
    {
        await Task.Delay(1000);
        _currentState = State.Connecting;
    }
}
```

Да и на самом то деле я сам не ведал что творил, знаний о FSM было практически ноль. Тяжело было изучать эту тему, слишком много информации и в рамках system design, и нововведений c# (даже новая идея появилась для блога: Class, Struct, Record)

Ознакомился про базовую спецификацию для FSM и несколько путей реализации, включая сам паттерн State.
Выучил что у классов состояния по большей части замыленные и в абстрактном мире. Этого не должно быть. В рамках разработки я, беря яблоко в руки, должен не только описывать методы взаимодействия с ним, но и явно указывать его состояния, тогда разработчик маловероятно будет писать код:
```cs
var myApple = new Apple();
var myNPC = new NPC();
var rottenApple = myApple.TimeOfExist += 1 year;
myNPC.Heal(rottenApple);
```