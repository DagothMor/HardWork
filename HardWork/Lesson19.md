# Файл в кеше и его метадата
*Допустим для оптимизации отслеживания файлов в какой нибудь папке* мы создаем кеш, являющийся конкуррентным словарем, где ключ - строка путь к файлу, а значение - метадата документа.
Если программа впервые сталкивается с файлом, которого в кеше не существует, она пытается открыть файл, и прочитать его метаданные.
Как происходит добавление:
```cs
        public void Add(string filePath, long PID, MetadataDto metadata, bool isOpened = false)
        {
            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileDataCache)}.{nameof(Add)}. ->");

            if (String.IsNullOrEmpty(filePath) || metadata == null)
            {
                //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileDataCache)}.{nameof(Add)}. filePath or metadata is null.");
                return;
            }
	    cache.AddOrUpdate(filePath,PID,metadata,isOpened) == null;
		//...
```
Во время параллельной разработки над этим проектом, из за отсутствия документации и согласований были разные виды кешей, например кеш открытых документов, кеш для мониторинга печати, кеш для создающихся документов, глобальный, отвечающий за хранение метаданных во время круд операций на уровне проводника. В кеше печати разработчик указал ключ как гуид из метадаты файла. И правда, гуид файла по сути может являться первичным ключом поскольку уникален. Однако это шло в разрез с логикой глобального кеша, который говорит нам что ключ - полное имя файла.
Чтобы избавиться от путаницы в рамках программы объеденим все поля относящиеся непосредственно к файлу в единую структуру BaseCacheElement 
```cs
public BaseCacheElement(DateTime ttl, MetadataDto metadata, string filename, string fullPath, long PID, bool isOpened)
        {
            _ttl = ttl.AddSeconds(30);
            _metadata = metadata;
            _fileName = filename;
            _fullPath = fullPath;
            _PID = PID;
            _isOpened = isOpened;
        }
```

Теперь же элемент в словаре является immutable. Если мы хотим изменить значение то нам нужно будет пересоздавать объект с новыми параметрами.
```cs
public void Add(string filePath, long PID, MetadataDto metadata, bool isOpened = false)
        {
            //Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileDataCache)}.{nameof(Add)}. ->");

            if (String.IsNullOrEmpty(filePath) || metadata == null)
            {
                //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileDataCache)}.{nameof(Add)}. filePath or metadata is null.");
                return;
            }
            
OpenedFileCacheElement openedFileCacheElement = new OpenedFileCacheElement(
                ttl: DateTime.UtcNow.AddSeconds(30),
                metadata: metadata,
                filename: Path.GetFileName(filePath),
                fullPath: filePath,
                PID: PID,
                isOpened: isOpened);
            cache.TryRemove(filePath,out _);
            if (cache.AddOrUpdate(openedFileCacheElement) == null)
            {
                //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(FileDataCache)}.{nameof(Add)}. failed adding in cache {filePath}");
            }
```
В рамках хранения событий в базе данных такая же ситуация. Есть таблицы:
- Файл(полный путь к файлу, путь к родительскому файлу, дата создания, внешний ключ на таблицу метадата...), 
- Метадата(гуид документа,гуид родительского документа,метка...)
- Событие(файл,тип события,время, внешний ключ на пользователя)
- Пользователь(имя, SID)

Сложно подсчитать сколько раз 1 человек за время своей работы может отправить событий на сервер, еще сложнее представить сколько выйдет за день со штатом в 1000 человек

и если таблицы file и metadata одинаковы по существу(связь 1 к 1, обе описывают одну и ту же абстрактную единицу) и их можно денормализовать, обьеденив в 1 таблицу, то с событием и пользователем посложнее, да объединить их можно, тогда не нужно будет использовать операцию join(например для выяснения кто пытался больше всего распечатать документы с коммерческой тайной.), но и обьем базы расширится.