
# 1.1 
Вернемся к ManagerOfFile.
Напомню что данный класс должен читать метадату документа и менять/создавать ее.
Посмотрим на интерфейс
```cs
	/// <summary>
    /// Управление файлом.
    /// </summary>
    public interface IManagerOfFile : IDisposable
    {
        /// <summary>
        /// Текущие метаданные в памяти.
        /// </summary>
        MetadataDto Metadata { get; set; }

        bool SetReadOnly(List<MarkerDto> availableMarks);

        /// <summary>
        /// Изменить метадату шаблона для будущего нового документа.
        /// </summary>
        /// <returns>Успех</returns>
        bool ResetMetadataForTemplateDocument();


        /// <summary>
        /// Делает текущий гуид родительским и генерирует новый.
        /// </summary>
        /// <param name="parentGuid">Родительский гуид</param>
        /// <returns>Успех</returns>
        bool CreateMetadataForChild(out MetadataOfDocument metadataOfDocument);

    }
```
Из за наработок с разных веток произошла путаница и неразбериха. Сам документ закрывается тогда когда вызывается Dispose класса. Однако появились методы которые взаимодействуют с атрибутами документа или же меняют метадату закрывая после всех операций сам документ, отчего он при возможных дальнейших действий может вызвать ошибку. Идеальный вариант это избавление в конструкторе/деструкторе открытие - закрытие документа и свойств(геттера и сеттера) и написание методов с явным открытием и закрытием файла.

___
# 1.2
Помню пару лет назад наткнулся на очень интересный баг в HOTA, когда играл в XL карту. Враг в виде бота использовал толи крылья левитации, толи сандали хождения по воде, в общем он остановил свой ход прямо на воде... В недоумении, я, использовав телепорт в город, ближайший к этому врагу, использовал сам хождение по воде. По правилам генерации хода, ты можешь пройти по воде, но только если твой ход заканчивается на суше. Я же смог заставить персонажа атаковать врага стоящего прямо на воде. По итогу выскочило окошко обмена войском/артефактами с самим собой :)
Попробуем описать Действия
```cs
public interface TurnGeneration {
    void makeMove();
    void makeAttack();
    void passThroughSea();
    void teleport();
}
```

Как возможный вариант исправления такой ошибки это создание состояний внутри класса и использования их для контроллирования выполнения методов сделать ход, совершить нападение, пройти сквозь воду, телепортироваться.
___
# 2.1
ProgramManager это класс который сопоставляет расширение документа с исполняемым файлом(где по умолчанию первый элемент в списке).
Код до
```cs
public ProgramManager()
        {
            DefalutProgram = new Dictionary<string, LinkedList<string>>(){
            {"docx", new LinkedList<string>( new List<string> { "word", "librewriter", "myofficetext", "r7office" }) },
            {"odt", new LinkedList<string>( new List<string> { "word", "librewriter", "myofficetext", "r7office" }) },
            {"doc", new LinkedList<string>( new List<string> { "word", "librewriter", "myofficetext", "r7office" }) },

            {"xlsx", new LinkedList<string>( new List<string> { "myofficeexcel", "librecalc", "myofficenf", "kbwf", "r7office" }) },
            {"ods", new LinkedList<string>( new List<string> { "myofficeexcel", "librecalc", "myofficenf", "kbwf", "r7office" }) },
            {"xls", new LinkedList<string>( new List<string> { "myofficeexcel", "librecalc", "myofficenf", "kbwf", "r7office" }) },

            {"pptx", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "myofficepresentation", "r7office" }) },
            {"odp", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "myofficepresentation", "r7office" }) },
            {"ppt", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "myofficepresentation", "r7office" }) },

            {"pdf", new LinkedList<string>( new List<string> {"Acrobat Rider", "FoxitReader" }) },

            {"vsd", new LinkedList<string>( new List<string> {"visio" }) },
            {"vsdx", new LinkedList<string>( new List<string> {"visio" }) }
        };
```

Код после
```cs
public ProgramManager(Dictionary<string, LinkedList<string>>() programRule)
{
	DefalutProgram = programRule;
}
```

Что по итогу хуже, программа которая упала при инициализации из за сломанного формата json в config файле, или же скрытый баг запуска ворда, хотя пользователь ожидал либру? Второе, поскольку последние логи остановятся именно на попытке десериализации конфигурационного файла и будет сразу же понятна ошибка, в отличии от того что тебе дают простыню логов и попробуй найти тот участок в котором ProgramManager не запустился так как надо.

___
# 2.2
Допустим мы ввели донаты для нашей ммо и теперь, игроки купившие подписку могут получить бонус здоровья и маны на старте игры.
```cs
public class Orc
{
    public int HealthPoints { get; set; }
    public int Mana { get; set; }

    // Пустой конструктор
    public Orc()
    {
    }

    // Конструктор с параметрами
    public Orc(int healthPoints, int mana)
    {
        HealthPoints = healthPoints;
        Mana = mana;
    }
}
```
Самый плохой случай когда оставили пустым конструктор, может возникнуть баг что любой игрок, платил он или нет, на старте игры сразу же помрет(дефолтное значение здоровья 0), а все потому что кто-то обязательно забудет/не вспомнит/даже не знает о том что пустой конструктор не имеет явного определения для полей.
Исправим
```cs
public class Orc
{
    public int HealthPoints { get; set; }
    public int Mana { get; set; }

    // Пустой конструктор
    public Orc()
    {
	    HealthPoints = 100;
	    Mana = 100;
    }

    // Конструктор с параметрами
    public Orc(int healthPoints, int mana)
    {
        HealthPoints = healthPoints;
        Mana = mana;
    }
}
```
Вроде получше, но вот беда, донатеры генерят своих персонажей с базовыми здоровьем/маной хотя мы явно обещали бонусы, а все потому что кто-то обязательно забудет/не вспомнит/ не заметит вызова пустого класса в котором поля по умолчанию.
Решение очевидное, оставить конструктор в котором определяются все поля. Вызывать же конструктор там где мы уже получили будущие значения полей персонажа.
```cs
if (player.LicenseType == "Бесплатная")
{
    Orc orc = new Orc(100, 100);
    return;
}
if (player.LicenseType == "VIP")
{
    Orc orc = new Orc(120, 120);
    return;
}
```
___
# 3.1
Явным примером будет создание перечисления DocumentExtensions и OfficePrograms для работы с получением расширения и исполняемого файла для него.
```cs
	public enum DocumentExtensions
    {
        docx,
        pdf,
        odt,
        //...
    }
    public enum OfficePrograms
    {
        word,
        myofficetext,
        r7office,
        //...
    }
```
На самом деле легче не потому что сложнее ошибиться/опечататься при работе  с расширением, а потому что не надо вспоминать ключевые слова триггеры которые мы передаем в аргументы приложения.
___
# 3.2
Представим ситуацию когда мы разрабатываем симулятор свиданий, нам нужно помочь Кеке завоевать сердце Бабы
```cs
// Baba Is Hot
if (answer == 1)
{
    attractivenessRatingBabas -= AttractivenessRating.VeryBad;
    return;
}
// Kiss Is Win
if (answer == 2)
{
    attractivenessRatingBabas -= AttractivenessRating.Bad;
    return;
}
// Keke Has Choco And Flower
if (answer == 3)
{
    attractivenessRatingBabas -= AttractivenessRating.Neutral;
    return;
}
// Keke Has Float
if (answer == 4)
{
    attractivenessRatingBabas -= AttractivenessRating.Good;
    return;
}
// Heart And Keke Is Melt
if (answer == 5)
{
    attractivenessRatingBabas -= AttractivenessRating.VeryGood;
    return;
}
```

Будет крайне удобно использовать перечисление вместо чисел для начисления очков привлекательности. Специально ограничиваем себя ради большего контроля, ведь что если мы начнем прибавлять +2, +100... получится что Baba Is Mad.
___
Данные примеры являются средством борьбы против главного врага программиста - его же самого, чем уже простор действий, тем более предсказуемое поведение.