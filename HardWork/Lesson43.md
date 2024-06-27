Задали хорошую задачу на собеседовании:

Вот оригинальный класс `CalendarService`, который определяет, является ли текущий день рабочим:

```cs
public class CalendarService
{
    public bool IsWorkingDay()
    {
        switch (DateTime.Now.DayOfWeek)
        {
            case DayOfWeek.Saturday:
            case DayOfWeek.Sunday:
                return false;
            default:
                return true;
        }
    }
}
```

Также у нас есть тест для этого класса:

```cs
public class CalendarServiceTests
{
    [Fact]
    public void IsWorkingDay_Always_ReturnsTrue()
    {
        var calendarService = new CalendarService();
        
        var actualDay = calendarService.IsWorkingDay();
        
        Assert.True(actualDay);
    }
}
```

Потому в CICD бага такая, что в выходные тест падает, очевидно, а в выходные должен билдится и накатывается прод. Потому нужно найти решение.

А решение такое что нужно создать интерфейс `IDateTimeProvider`, который будет предоставлять текущую дату и время. Затем создать реализацию этого интерфейса и зарегистрировать их в контейнере зависимостей. В тестах теперь мы сможем замокать этот интерфейс, чтобы контролировать возвращаемые значения.

```cs
public interface IDateTimeProvider
{
    DateTime GetNow();
}
```

```cs
public class DateTimeProvider : IDateTimeProvider
{
    public DateTime GetNow()
    {
        return DateTime.Now;
    }
}
```

Изменим `CalendarService`, чтобы использовать зависимость через конструктор:

```cs
public class CalendarService
{
    private readonly IDateTimeProvider _dateTimeProvider;

    public CalendarService(IDateTimeProvider dateTimeProvider)
    {
        _dateTimeProvider = dateTimeProvider;
    }

    public bool IsWorkingDay()
    {
        switch (_dateTimeProvider.GetNow().DayOfWeek)
        {
            case DayOfWeek.Saturday:
            case DayOfWeek.Sunday:
                return false;
            default:
                return true;
        }
    }
}
```

зарегистрируем интерфейс и его реализацию:

```cs
services.AddSingleton<IDateTimeProvider, DateTimeProvider>();
services.AddTransient<CalendarService>();
```

```cs
public class CalendarServiceTests
{
    [Fact]
    public void IsWorkingDayWeekdayReturnsTrue()
    {
        var mockDateTimeProvider = new Mock<IDateTimeProvider>();
        mockDateTimeProvider.Setup(m => m.GetNow()).Returns(new DateTime(Любой рабочий день));
        var calendarService = new CalendarService(mockDateTimeProvider.Object);
        var actualDay = calendarService.IsWorkingDay();
        Assert.True(actualDay);
    }
    
    [Fact]
    public void IsWorkingDayWeekendReturnsFalse()
    {
        var mockDateTimeProvider = new Mock<IDateTimeProvider>();
        mockDateTimeProvider.Setup(m => m.GetNow()).Returns(new DateTime(Любой выходной));
        var calendarService = new CalendarService(mockDateTimeProvider.Object);
        var actualDay = calendarService.IsWorkingDay();
        Assert.False(actualDay);
    }
}
```

Тут по большей части решение в DI, но моки все равно используются.

Абстрактный эффект: Проверка буднего дня

Явность в коде: Тест явно проверяет день, используя мок.

# 2.2
https://code-maze.com/dotnet-unit-testing-mock-file-system/

```csharp
[TestMethod]
public void TestReadFile()
{
    var result = FileReader.ReadFile("test.txt");
    Assert.AreEqual("File content", result);
}
```

Тест зависит от содержимого файла и способа его чтения.

```csharp
using Moq;
using System.IO.Abstractions;
using System.IO.Abstractions.TestingHelpers;

[TestMethod]
public void TestReadFile()
{
    var mockFileSystem = new MockFileSystem();
    mockFileSystem.AddFile("test.txt", new MockFileData("File content"));

    var fileReader = new FileReader(mockFileSystem);
    var result = fileReader.ReadFile("test.txt");

    Assert.AreEqual("File content", result);
}
```

Абстрактный эффект: Чтение содержимого файла.

Явность в коде: Тест явно проверяет содержимое файла, используя мок файловой системы.
# 4
Моки не использовал, потому что по большей части как было сказано в предыдущем ответе, тестированием занимаются аналитики(отладка бизнес процессов над документами, сам проект жестко связан с внутренней закрытой апи компании).