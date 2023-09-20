___
# 2.1. Класс слишком большой (нарушение SRP), или в программе создаётся слишком много его инстансов (подумайте, почему это плохой признак).

Класс DriverMessageHandler представляет собой обработчик сообщений от драйвера файловой системы и генерация для него ответа, в нем:
24 поля
20 методов
2000 строк кода

Сначала происходит фильтрация сообщения по пайплайну
```
// Пропустить обработку сообщения, если процесс, который обращается к файлу из списка необрабатываемых.
...
// Пропустить обработку сообщения, если файл шаблонный.
...
// Пропустить обработку сообщения, если файл из пути для шаблонов.
...
// Пропустить обработку сообщения, если файл является исполняемым.
...
// Пропустить обработку документов неподдерживаемых форматов.
...
// Заблокировать доступ процессу, не предназначенному для обработки документов поддерживаемых форматов.
...

```
После успешной фильтрации нам нужно обработать сообщение в зависимости от PID процесса(логика для explorer полностью отличается от WINWORD.EXE). Все это так же вызывается в нашем пайплайне

Внутри обработки отправляется в потокобезопасную очередь отправки сообщений. Сама очередь и логика отправки находится так же в DriverMessageHandler.

Я уже задумывался над этой проблемой, ибо для того чтобы отправить одно из событий(В тот момент оно ловилось тогда когда PID нашего процесса закрывался), приходилось пробрасывать DriverMessageHandler в конструктор класса ProcessMonitor, который в отдельном потоке подписывается на событие удаления с помощью системного класса ManagementEventWatcher. Получается так что пробрасываем целый класс только для того чтобы отправить событие на сервер, и сделано это было в рамках исследования, но нет ничего постояннее чем временное...

Решение в очевидной декомпозиции. Если класс обрабатывает ответ то он только это и должен делать. Логика получения событий в потокобезопасную очередь, чтение и отправка должна быть написана отдельным классом EventSender.

___

# 2.2
Код был создан для получения имени процесса и его родительского PID
Код до
```cs
	/// <summary>
    /// Имя процесса.
    /// </summary>
    public class ProcessName
    {
        /// <summary>
        /// Получить имя процесса.
        /// </summary>
        /// <param name="longPid">ID процесса.</param>
        /// <returns>Имя или Empty, если ошибка.</returns>
        public static string GetName(long longPid)
        {
            if (longPid == -1) return String.Empty;
            System.Diagnostics.Process proc = null;
            int pid = 0;
            try
            {
                pid = Convert.ToInt32(longPid);
                proc = System.Diagnostics.Process.GetProcessById(pid);
                return proc.ProcessName;
            }
            catch (Exception e)
            {
                //Logger.Error($"Error get process with Pid: {longPid}. Error: {e}");
                return String.Empty;
            }
        }
        /// <summary>
        /// Получить PID родительского процесса.
        /// </summary>
        /// <param name="longPid">ID процесса.</param>
        /// <returns>Имя или null, если ошибка.</returns>
        public static int GetParentPID(long longPid)
        {
            System.Diagnostics.Process childProc = null;
            int childPid = 0;
            try
            {
                childPid = Convert.ToInt32(longPid);
                childProc = System.Diagnostics.Process.GetProcessById(childPid);
                System.Diagnostics.Process parProc = ParentProcessUtilities.GetParentProcess(Int32.Parse(childPid.ToString()));
                //Logger.Trace($"#### Child Pid: {childPid}. Child name: {childProc.ProcessName}");
                if (parProc is null)
                {
                    return -1;
                }
                //Logger.Trace($"Parent name: {parProc.ProcessName}");
                return parProc.Id;
            }
            catch (Exception e)
            {
                //Logger.Error($"Error get process with Pid: {longPid}. Error: {e}");
                return -1;
            }
        }
    }
```

Данная логика была перенесена в класс ProcessNameManager, потому что семантика схожа.

# 2.3

В классе по отправке событий для клонируемых файлов был метод сравнивающий имена родительского и итогового файла, потому что родительское имя могло измениться при копировании множества документов(глобальные переменные - зло).
```cs
		/// <summary>
        /// Является ли родительский документ первичным клонируемым для дочернего, например:
        /// Док - копия и Док - копия(N)
        /// </summary>
        /// <param name="parentFilePath"></param>
        /// <param name="childFilePath"></param>
        /// <returns></returns>
        private bool FilePathsAreCloning(string parentFilePath, string childFilePath)
        {
            try
            {
                if (String.IsNullOrEmpty(parentFilePath))
                {
                    //Logger.Error($"String.IsNullOrEmpty(parentFilePath).");
                    return false;
                }
                if (String.IsNullOrEmpty(childFilePath))
                {
                    //Logger.Error($"String.IsNullOrEmpty(childFilePath).");
                    return false;
                }
                //Logger.Trace($"Checking childFileCopyWord:{childFilePath}: parentFileCopyWord{parentFilePath} are same.");
                // как минимум у родителя —_копия а у дочернего файла —_копия (N)
                if (parentFilePath.Length < 7 || childFilePath.Length < 11)
                {
                    //Logger.Error($" parentFilePath.Length < 7 || childFilePath.Length < 11.");
                    return false;
                }
                // но может быть что у дочернего —_копия (NNN...)
                int countOfDigitsInParenthesis = 0;
                var stackOfReversedDigits = new Stack<Char>();
                if (childFilePath[childFilePath.Length - 1] != ')')
                {
                    //Logger.Error($"childFilePath[childFilePath.Length - 1] != ')'.");
                    return false;
                }
                for (int childFilePathLetter = childFilePath.Length - 2; childFilePathLetter > 0; childFilePathLetter--)
                {
                    if (childFilePath[childFilePathLetter] == '(')
                    {
                        break;
                    }
                    if (Char.IsDigit(childFilePath[childFilePathLetter]))
                    {
                        countOfDigitsInParenthesis++;
                        stackOfReversedDigits.Push(childFilePath[childFilePathLetter]);
                        continue;
                    }
                    // не может такого быть —_копия && —_копия (123a4)
                    {
                        //Logger.Error($"childFilePath[childFilePathLetter] == {childFilePath[childFilePathLetter]}");
                        return false;
                    }
                }
                // не может такого быть —_копия && —_копия ()
                if (countOfDigitsInParenthesis == 0)
                {
                    //Logger.Error($"countOfDigitsInParenthesis == 0");
                    return false;
                }
                // Вытаскиваем из стека в билдер получая правильный порядок цифр
                var bufferOfDigits = new StringBuilder();
                while (stackOfReversedDigits.Count > 0)
                {
                    bufferOfDigits.Append(stackOfReversedDigits.Pop());
                }
                string digitsInParenthesis = bufferOfDigits.ToString();
                bufferOfDigits = null;
                if (!Int32.TryParse(digitsInParenthesis, out _))
                {
                    // не может такого быть —_копия (2147483647)
                    return false;
                }
                // если не хватает NumberBuffer.Length символов _(NNN...) до равенства копий
                var LengthOfCopyingFilesAreEqual = parentFilePath.Length + countOfDigitsInParenthesis + 3 == childFilePath.Length;
                if (!LengthOfCopyingFilesAreEqual)
                {
                    //Logger.Error($"LengthOfCopyingFilesAre NOT Equal");
                    return false;
                }
                if (childFilePath[childFilePath.Length - 2 - countOfDigitsInParenthesis] != '(')
                {
                   //Logger.Error($"childFilePath[childFilePath.Length - 2 - countOfDigitsInParenthesis] != '('");
                    return false;
                }
                if (childFilePath[childFilePath.Length - 3 - countOfDigitsInParenthesis] != ' ')
                {
                    //Logger.Error($"childFilePath[childFilePath.Length - 3 - countOfDigitsInParenthesis] != ' '");
                    return false;
                }
                var childFileCopyWord = childFilePath.Substring(childFilePath.Length - 10 - countOfDigitsInParenthesis, 7);
                var parentFileCopyWord = parentFilePath.Substring(parentFilePath.Length - 7, 7);
                //Logger.Trace($"childFileCopyWord:{childFileCopyWord}: parentFileCopyWord{parentFileCopyWord} are same.");
                return String.Equals(childFileCopyWord, parentFileCopyWord);
            }
            catch (Exception ex)
            {
                //Logger.Error($"{ex}");
                return false;
            }
        }
```
Данный метод был перенесен в статический класс FilePathManager, который будет отвечать за пути для файлов и их сравнение, являются ли в одном диске, в одной папке, одинаково ли названы итд...

# 2.4
Для избегания постоянного открытия документа и чтение его метаданных использовалась переменная в DriverMessageHandler именуемая MetadataCache, которая очищается с определенным интервалом из настроек конфигураций(отчего сыграло с нами злую шутку, был момент когда во время печати сгорал кеш метаданных(время хранения иссякало)). Потому было два варианта решения задачи, правильная в виде создания отдельного потокобезопасного класса в виде списка элементов в которых хранится сама метадата, путь к файлу и булева IsOpen(нельзя удалять элемент с метадатой хотябы в тот момент когда он открыт в приложении). Неправильная же состояла в том чтобы просто увеличить в конфигурации время жизни кеша. Угадайте какой мы выбрали в рамках "главное быстро сделать и показать"? :)

# 2.5
Несоблюдение принципа инверсии зависимостей и внедрения зависимостей.
Банальный пример с испольнованием логера
код до
```cs
public class ProductService
{
    private Logger logger = new Logger();

    public void SaveProduct(Product product)
    {
        // Логика сохранения продукта
        logger.Trace("Продукт сохранен");
    }
}
```

код после
```cs
public interface ILogger
{
    void Trace(string message);
}

public class FileLogger : ILogger
{
    public void Trace(string message)
    {
        // 
    }
}

public class FileManager
{
    private ILogger logger;

    public FileManager(ILogger logger)
    {
        this.logger = logger;
    }

    public void SaveFile(Product product)
    {
	    //...
        //Logger.Trace("Файл сохранен");
        //...
    }
}
```

Таким образом мы смогли ослабить зависимости. Теперь мы можем поменять конкретную реализацию ILogger не меняя ничего как в FileManager, так и в остальных классах.
# 2.6
На старой работе при отправке задачи на согласование по регламенту, подписывающим являлся род класс Recipient, он высчитывался от дочерних Employee(Employee-User-Recipient) - сотрудник или User(User-Recipient) - пользователь(контрагент). И в схеме был блок для подписания, и именно там было приведение типа подписывающего и в зависимости от типа выполнялись разные действия. Как вариант не приводить к Recipient, унаследовать род класс интерфейс IRecipient. Указать в конструкторе класса задачи не класс исполнителя, а именно интерфейс. Таким образом Задача стартует, в блоке подписывания явно сделать варианты для сотрудника и контрагента.
# 2.7
В рабочих проектах не встречалось, постараюсь развить мысль на любимом DwarfFortress.
Руководство решило выпустить новое масштабное обновление чтобы порадовать своих игроков новым сеттингом, а именно зомби.
У нас есть много классов в виде дворфов, собак, кошек...каштанового дерева.
Самый плохой вариант это наследоваться от классов выше и переписывать/расширять логику (DwarfZombie,DogZombie...), да и как так тогда реализовать логику заражения? Удалять объект дворфа и генерировать на основе его полей нового? Переносить всю историю его любви к табуреткам будет проблематично.
Как вариант вижу использование паттерна Декоратор. Создаем класс ZombieDecorator 
```cs
public abstract class CharacterDecorator : Character
{
    protected Character character;

    public CharacterDecorator(Character character)
    {
        this.character = character;
    }
}
public class ZombieDecorator : CharacterDecorator
{
    public ZombieDecorator(Character character) : base(character) { }

    public override string Say()
    {
        return character.Say() + "О кстати у тебя есть лишние мозги? Друг спрашивает...";
    }
}
```

Уверен ответ не очень оптимальный но как ни крутил, не смог придумать ничего более. Ведь если происходит ситуация когда нужно создать наследника, а по итогу и у остальных приходится это делать, то что это может означать? Когда ситуация А->А1 B->B1 C->C1 то не лучше ли будет рассмотреть 1 как явление отдельным классом или набором методов?  
# 2.8
Наследники должны расширять но не переопределять методы родительского класса, иначе наследование будет не истинным. Решается паттерном посетитель.
Код до
```cs
class Animal
{
    public virtual void MakeSound()
    {
        Console.WriteLine("Animal makes a sound.");
    }
}

class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Bark.");
    }
}
class Cat : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("Meow.");
    }
}
```
Код после
```cs
class Animal
{
    public virtual void Accept(AnimalVisitor visitor)
    {
        visitor.Visit(this);
    }
}

class Dog : Animal
{
    public override void Accept(AnimalVisitor visitor)
    {
        visitor.Visit(this);
    }
}

class Cat : Animal
{
    public override void Accept(AnimalVisitor visitor)
    {
        visitor.Visit(this);
    }
}

interface AnimalVisitor
{
    void Visit(Animal animal);
    void Visit(Dog dog);
    void Visit(Cat cat);
}

class SoundAnimalVisitor : AnimalVisitor
{
    public void Visit(Animal animal)
    {
        Console.WriteLine("Animal makes a sound.");
    }

    public void Visit(Dog dog)
    {
        Console.WriteLine("Bark.");
    }

    public void Visit(Cat cat)
    {
        Console.WriteLine("Meow.");
    }
}
```
Однако если посмотреть в рамках задания 3.2 то это будет считаться антипаттерном, ведь животинка издает звук? Да но по своему, сложность этого действия минимальна. Легче не думать и сделать за 5 минут чем подумать 5 минут и сделать, разница лишь в том что думать сможешь меньше на более важные вещи. Сам смысл паттерна скорее в применении более серьезной и разносторонней логики под одинаковым действием. 
# 3.1
Раньше метод по отправке сообщений принимал ряд параметров для файла.
Возникала путаница когда функционал дорос до того чтобы отправлять уведомление на сервер с гуидом родителя, приходилось во многих местах явно создавать новые переменные которых недоставало для метода отправки события файла.
Решением же стало выделение в отдельный EventFile класс, добавление полей и их последующее заполнение находится теперь только в конструкторе, ибо все нужные нам значения находятся в классе метадаты документа.
Даже если параметров много и все они разного типа, то нужно подняться на уровень меты выше, ибо они могут описывать 1 сущность с которой работать гораздо удобнее.

# 3.2 
Допустим мы реализовываем клон To-do, а следовательно создаем систему по управлению списком задач. Используем паттерн Команда
```cs
public class Task
{
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsCompleted { get; set; }
}
public class CreateTaskCommand
{
    private TaskList _taskList;

    public CreateTaskCommand(TaskList taskList)
    {
        _taskList = taskList;
    }

    public void Execute(string title, string description)
    {
        _taskList.AddTask(new Task { Title = title, Description = description });
    }
}

public class ChangeTaskStatusCommand
{
    private TaskList _taskList;

    public ChangeTaskStatusCommand(TaskList taskList)
    {
        _taskList = taskList;
    }

    public void Execute(Task task, bool isCompleted)
    {
        _taskList.UpdateTaskStatus(task, isCompleted);
    }
}
```
Но какой смысл если можно написать и без паттерна
```cs
public class Task
{
    public string Title { get; set; }
    public string Description { get; set; }
    public bool IsCompleted { get; set; }
}

public class TaskList
{
    private List<Task> _tasks = new List<Task>();

    public void AddTask(Task task)
    {
        _tasks.Add(task);
    }

    public void UpdateTaskStatus(Task task, bool isCompleted)
    {
        task.IsCompleted = isCompleted;
    }

    public void RemoveTask(Task task)
    {
        _tasks.Remove(task);
    }
}
```
Данный пример говорит не о SOLID как в предыдущих заданиях, а KISS