# 1

Старый заброшенный проект, который я делал до устройства на первую работу. Закрепление информации через дидактические карточки с таймингом по кривой Эббенгауза.
https://github.com/DagothMor/CogniMnemo/tree/develop

Архитектура - MVC, где View - работа с консолью, логика находится в папке Menus, Controller - вся логика над и с карточками, находится в папке Controllers.
Модель на данный момент только карточка.

```cs 
public class Card
	{
		public int Id { get; set; }
		// Дата создания
		public DateTime DateOfCreation { get; set; }
		// Дата последнего вызова
		public DateTime DateOfLastRecall { get; set; }
		// Дата следующего вызова
		public DateTime DateOfNextRecall { get; set; }
		// Уровень припоминания, чем меньше - тем чаще нужно припоминать.
		public byte Level { get; set; }
		// Вопрос
		public string Question { get; set; }
		// Ответ
		public string Answer { get; set; }

		//...
	}
```
База данных максимально простая - в папке по местоположению исполняемого файла создается/открывается папка database, и все карточки инициализируются в память приложения, через txt файлы.

Программа исполняла свои обязанности, однако нужно заняться рефакторингом, поскольку выдуманный заказчик посчитал что было бы неплохо прикрутить решение и для веба, и для телеграм бота, и для десктопа(на разные ОС с помощью open source приложения Avalonia), и локальное развертывание...Возможных требований много и потому наше приложение должно быть максимально гибким и безболезненно расширяемым.

Рассмотрим точку входа:
```cs
		static void Main(string[] args)
		{
			MainMenu.Start();
		}
		public static class MainMenu
	{
		public static void Start()
		{
			// TEST: Удаляем все карты для тестирования функционала.
			// ОЧИЩАЕТ ПАПКУ БАЗЫ ДАННЫХ ОТ ВСЕХ СОЗДАННЫХ КАРТОЧЕК.
			DataBaseInitialization.DeleteAllCardsInDataBase();
			// Заполняем базу данных 
			MockController.CreateListOfTemplateCards();
			//todo:создать уведомление для ос о том что пора вспомнить карту(отталкиваясь от аттрибутов Last recall и level)
			// TODO: Добавить функцию очищения корзины.
			while (true)
			{
				Console.WriteLine($"Welcome {Environment.UserName} to the CorgiMneemo!" + Environment.NewLine +
					"The best application for training your skills!");
				Console.WriteLine("Main menu:" + Environment.NewLine +
					"1-Training menu" + Environment.NewLine +
					"2-Card menu." + Environment.NewLine +
					"3-Options" + Environment.NewLine +
					"4-Help" + Environment.NewLine +
					"exit-exit from application");
		//...
```
На момент написания прошло 1000 дней, как быстро летит время...

В Main методе происходит Вызов сценария главного меню, банальная ошибка в том что инициализация во время вызова домашнего ui. Мы еще не говорили о полном отсутствии логирования, возможном хуке от сервера для обновления базы данных карточек, вызов системного уведомления о том что N количество дидактических карт уже стоит припомнить на данный момент...

Конечно же решением будет добавление внедрения зависимостей.
```cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;

public class Program
{
    public static void Main(string[] args)
    {
        var serviceProvider = new ServiceCollection()
            .AddLogging()
            .AddSingleton<IConsoleService, ConsoleService>()
            .BuildServiceProvider();

        //configure console logging
        serviceProvider
            .GetService<ILoggerFactory>()
            .AddConsole(LogLevel.Debug);

        var logger = serviceProvider.GetService<ILoggerFactory>()
            .CreateLogger<Program>();
        logger.LogDebug("Start application.");

        var consoleService = serviceProvider.GetService<IConsoleService>();
        consoleService.Start();
        logger.LogDebug("End application.");
    }
}
```
# 2
Однако возникает вопрос на уровень выше, как хранить и обрабатывать базу данных? Точно ли карточка это просто вопрос - ответ - когда вспомнил/вспомнишь? Будет ли всегда текстом или может заказчик захочет работать с MarkDown? Как хранить файлы? Какую базу данных использовать, или же как сделать так чтобы было легко заменять саму базу данных? Будут ли данные храниться локально, или же отправляться на сервер?
Неизвестно, потому предоставим выбор пользователю. Предположим что во время установки мы запросим ip сервера, который будет хранить все дидактические карточки пользователя, если он не авторизован или у него нет учетной записи, или, если он желает хранить локально, то предоставим ему следующий выбор: Хранить всю картотеку в базе данных, или же списком файлов, который пользователь может редактировать в любом удобном для него редакторе. Сам выбор будет храниться в конфигурационном файле. И если работа с разными реляционными бд(postgres,sql server,mysql...) понятна посредством ORM, то как добавить функционал по работе с txt файлами?

Будем работать с базой данных, хранящейся непосредственно в памяти приложения.

```cs
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    if (isLocal)
    {
        // Подключение к базе данных
        optionsBuilder.UseSqlServer(connectionString);
    }
    else
    {
        // Использование файла
        optionsBuilder.UseInMemoryDatabase("MyEntities");
    }
}

```

Инициализация будет происходить в конструкторе класса.
```cs
class MyContextInitializer : DropCreateDatabaseAlways<MyContext>
{
    protected override void Seed(MobileContext db)
    {
        Logger.Trace("Initialize a database...");
		DataBaseInitialization.Start(MyContext);
    }
}
```
Таким образом мы смогли сохранить гибкость работы с хранилищем данных в виде обычной базой данных, так и с дружелюбным для пользователя вариантом в виде множества файлов, которые могут храниться локально и легко модифицироваться.
# 3 

Как же высчитывалось новое время для карточки которую мы припоминали?
```cs
while (true)
				{
					Console.WriteLine("is your answer correct? Y/N");
					var userinput = Console.ReadLine();
					if (userinput.ToLower() == "y")
					{
						TextController.RewriteTextAfterAnInputAttribute(oldercard.Id, "date of last recall", DateTime.Now.ToString() + Environment.NewLine);
						oldercard.DateOfNextRecall = EbbinghausCurve.GetTimeRecallByForgettingCurve(DateTime.Now, oldercard.Level, '+');
//...
```

Мы жестко привязаны к кривой Эббингауза, рассмотрим ее детальнее

```cs
public static class EbbinghausCurve
	{
		//																									TimeSpan(day,hour,minute,second)
		private static readonly Dictionary<byte, TimeSpan> _ebbinghausCurve = new Dictionary<byte, TimeSpan>() { { 0, new TimeSpan(0,0,5)}
																											, { 1, new TimeSpan(0,0,25) }
																											, { 2, new TimeSpan(0,2,0) }
																											, { 3, new TimeSpan(0,10,0) }
																											, { 4, new TimeSpan(1,0,0) }
																											, { 5, new TimeSpan(5,0,0) }
																											, { 6, new TimeSpan(1,0,0,0) }
																											, { 7, new TimeSpan(5,0,0,0) }
																											, { 8, new TimeSpan(30,0,0,0) }
																											, { 9, new TimeSpan(150,0,0,0) }
																											, { 10, new TimeSpan(720,0,0,0) }
																											, { 11, new TimeSpan(3600,0,0,0) }
																											, { 12, new TimeSpan(21600,0,0,0) }
		};
		//todo:maybe need instead of dateoflastrecall just a datetime.now?
		public static DateTime GetTimeRecallByForgettingCurve(DateTime dateOfLastRecall, byte level, char plusorminus)
		{
			return plusorminus=='+' ? dateOfLastRecall += _ebbinghausCurve[level]: level!=0?dateOfLastRecall += _ebbinghausCurve[level--]: dateOfLastRecall += _ebbinghausCurve[0];
		}
	}

```

грамотным решением будет инвертировать зависимость, а так же почистить код.
Ограничим уровни с отрицательными значениями, заменим + - булевой БылаЗапомнена
```cs
public interface IForgettingCurve
{
    DateTime GetTimeRecallByForgettingCurve(DateTime dateOfLastRecall, UInt16 level, bool isMemorized)
}

public class EbbinghausCurveTimeRecallProvider : IForgettingCurve
{
    private static readonly Dictionary<UInt16, TimeSpan> _ebbinghausCurve = new Dictionary<UInt16, TimeSpan>()
    {
        { 0, new TimeSpan(0,0,5) },
        { 1, new TimeSpan(0,0,25) },
        { 2, new TimeSpan(0,2,0) },
        { 3, new TimeSpan(0,10,0) },
        { 4, new TimeSpan(1,0,0) }
    };

    public static DateTime GetTimeRecallByForgettingCurve(DateTime dateOfLastRecall, UInt16 level, bool isMemorized)
		{
			return isMemorized 
				? dateOfLastRecall += _ebbinghausCurve[level]
				: level!=0
					? dateOfLastRecall += _ebbinghausCurve[level--]
					: dateOfLastRecall += _ebbinghausCurve[0];
		}
}
```

Остается добавить в DI
```cs
var serviceProvider = new ServiceCollection()
            .AddLogging()
            .AddSingleton<IForgettingCurve, EbbinghausCurveTimeRecallProvider>()
            .BuildServiceProvider();
```

Так мы смогли сделать настройку припоминания более гибкой, можно сделать гибче манипулируя со словарем, а именно уровнями припоминания и собирать статистику с людей занимающихся по модифицированной кривой забывания.

# UseCases

Главный актор: Пользователь

Первый сценарий:
1. Пользователь запускает приложение.
2. Приложение отображает главное меню.
3. Пользователь выбирает опцию "Добавить карточку".
4. Приложение просит ввести вопрос
5. Пользователь вводит вопрос
6. Приложение просит ввести ответ
7. Пользователь вводит ответ
8. Приложение выводит введенный вопрос и ответ, спрашивая пользователя о корректности
9. Пользователь подтверждает
10. Приложение сохраняет карточку в базе данных и спрашивает пользователя хочет ли он добавить еще одну карточку
11. Если пользователь отказывается, переход к пункту 2, иначе в пункт 4 
12. Окончание первого сценария.

Второй сценарий:
1. Пользователь запускает приложение.
2. Приложение отображает главное меню.
3. Пользователь выбирает опцию "Начать припоминание".
4. Приложение ищет карточку, дата припоминания которой ближе всего к текущему времени, или та что больше всех просрочена
5. Если приложение не нашло ни одной карточки - уведомляет пользователя об отсутствии карточек для припоминания и переходит в пункт 2
6. Если карточка найдена то приложение показывает вопрос карточки пользователю
7. Пользователь пишет свой ответ на основе этой карточки
8. Приложение показывает ответ карточки и ответ пользователя, спрашивая его, корректно ли он ответил на основе ответа карточки
9. Если пользователь подтверждает то приложение увеличивает дату припоминания на уровень+ от карточки
10. Если пользователь указал что не смог корректно вспомнить то приложение уменьшает уровень карточки, регистрируя новую дату для припоминания
11. Приложение спрашивает хочет ли пользователь продолжить
12. Если пользователь соглашается то переход в пункт 4
13. Если пользователь отказывается то переход в пункт 2
14. Окончание второго сценария.