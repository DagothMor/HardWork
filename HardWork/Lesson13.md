# 1 Использование операторов объединения со значением null
Оператор объединения с NULL `??` возвращает значение своего операнда слева, если его значение не равно `null`. В противном случае он вычисляет операнд справа и возвращает его результат.

было
```cs
if (_settingsDto == null)
{
	_settingsDto = new SettingsDto();
} 
```
стало 
```cs
_settingsDto ??= new SettingsDto();
```

Дальше пример декоратора, если мы пытаемся обернуть null объект 
было
```cs
public sealed class Wrapper<T> where T : new()
{
    private T _source;
    
    public Wrapper(T source)
    {
	    if(source is null)
	    {
		    _source = new T();
		    return;
	    }
	    _source = source;
    }
}
```
стало
```cs
public sealed class Wrapper<T> where T : new()
{
    private T _source;
    public Wrapper(T source = null) => _source = source ?? new T();
}
```
Читается намного удобнее и последовательнее.
# 2 Правильный дизайн для избегания конструкций try catch
Давайте рассмотрим код, пытающийся открыть файл для записи метадаты шаблонного документа
До
```cs
while (true)
            {
                try
                {
                    if (countOfTryies >= 100)
                    {
                        //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(App)}.{nameof(CreateDocxScenario)}. ##### countOfTryies > 100 SOMETHING WAS WRONG");
                        return false;
                    }
                    IFileManager file = new FileManager(TEMPLATE_DOCX_PATH, _currentContext);

                    var success = file.ResetMetadataForTemplateDocument();
                    if (!success)
                    {
                        //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(App)}. ##### SOMETHING WAS WRONG.");
                        file.Dispose();
                        return false;
                    }
                    file.Dispose();
                    break;
                }
                catch (Exception ex)
                {
                    //Logger.Error($"ERROR: {ex}");
                    countOfTryies += 1;
                    Thread.Sleep(1000);
                    continue;
                }
            }
```

Как видим применение try catch очевидна, но не очевиден тот факт, что на самом деле мы отлавливаем только 1 исключение, а именно файл занят другим процессом.
Обязательно ли ловить одну и ту же ошибку? Может нам достаточно написать метод который будет ожидать освобождение файла, блокировать его и давать доступ к дальнейшим манипуляциям?
Код после
```cs
/// <summary>
        /// Изменить метадату шаблона для будущего нового документа.
        /// </summary>
        /// <returns>Success</returns>
        public bool ResetMetadataForTemplateDocument()
        {
            MarkerFactory markerFactory = new MarkerFactory();
            // Ждем когда файл заблокируется другим процессом.
            _document = markerFactory.WaitForOpen(_filePath);
            // При таймауте возвращаем false.
            if (_document == null)
            {
                //Logger.Error($"...");
                return false;
            }
            //...
        }
```

```cs
IFileManager file = new FileManager(TEMPLATE_DOCX_PATH, _currentContext);
			// Внутри нашего метода мы ожидаем освобождение файла, при таймауте мы будет возвращать false.
            var success = file.ResetMetadataForTemplateDocument();
            if (!success)
            {
                DSSLogger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(App)}. ##### SOMETHING WAS WRONG.");
                file.Dispose();
                return false;
            }
            file.Dispose();
```

# 3 Конструкция Using
Инструкция `using` обеспечивает правильное использование экземпляра [IDisposable](https://learn.microsoft.com/ru-ru/dotnet/api/system.idisposable)(Предоставляет механизм для освобождения неуправляемых ресурсов.)

При объявлении `using` в объявлении локальная переменная удаляется в конце область, в котором она объявлена. В предыдущем примере удаление происходит в конце метода .
Учитывая тот факт что закрытие файла мы перенесли в конец вызываемых методов, но никак не в Dispose(), то можем не беспокоится о подвешенном состоянии файла при дальнейшей отработке кода.
Код после
```cs
// Внутри нашего метода мы ожидаем освобождение файла, при таймауте мы будет возвращать false.
			// Для однострочных операций, можем убрать тело метода.
using IFileManager file = new FileManager(TEMPLATE_DOCX_PATH, _currentContext);
if (!file.ResetMetadataForTemplateDocument())
    return false;
//...        
```

Во время рефакторинга никогда не задумывался касаемо того а зачем мне ловить одну и ту же ошибку, если я могу подойти с другой стороны(проверка свободен ли файл) и не вызывать ее, избежав вызов громоздкой конструкции и понизив удобочитаемость.
Довольно интересно вышел для меня этот урок, так же напомнил себе об удобстве операторов для null, и вообще, ощутил прекрасное чувство когда смог сжать свой код до минимальных значений.