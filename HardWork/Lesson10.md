# 1 Слишком большой метод

При вызове метода отправки на сервер уведомления передавая в параметры путь к файлу, его родителя и тип события не учитывался тот факт, что в зависимости от типа событий мы должны произвести массу дополнительных действий. Обработчик может послать уведомление еще не создавшегося файла(событие копирования), или даже если он начал существовать, нужно изменить его метаданные(в частности создать новый идентификатор для дочернего файла, иначе может произойти путаница, изменен файл по  С/11/22//док.docx, а на сервере отобразится, что изменился файл С/11/док.docx).
Посмотрим на примере
## 1 до
```cs
/// <summary>
        /// Отправить сообщение-событие "клиента".
        /// </summary>
        /// <param name="eventMsg">Сообщение-событие для отправки на web сервер клиента.</param>
        /// @todo Переделать получение глобальные _guidMarker, _guidDoc, _usernameDoc для которых нет открытия
        /// службой файла.
        private void SendEventMessage(ClientEventMessage eventMsg)
        {
	        ///...
            /// Инициализация, проверка на Null, получение СИД пользователя.
            
            //____________________________________
            if (eventMsg.DocEvent == eEventType.CopyDocEvent)
            {
                if (String.IsNullOrEmpty(eventMsg.CloningFileName))
                {
                    return;
                }
                int countOfTryies = 0;
                while (true)
                {
                    try
                    {
                        MetadataDto newMetadataDTO = null;
                        var parentMetadata = _fileDataCache.TryGet(eventMsg.CloningFileName);
                        if (parentMetadata == null)
                        {
                            countOfTryies++;
                            Thread.Sleep(1000);
                            continue;
                        }
                        bool metadataAreEqual;
                        
                        // Чтобы избежать ошибки что документ не был открыт, открываем документ.
                        IFileManager firstFileGetMetadata = new FileManager(eventMsg.FileName, eventMsg.Context, needSaveToFile: true, needCreatePublicMarker: true);

                        var tempMetadata = firstFileGetMetadata.Metadata;
                        
                        metadataAreEqual = tempMetadata.Id == parentMetadata.Metadata.Id;
                        
                        // проставить булеву, чтобы не вызывали CreateMetadataForChild.
                        firstFileGetMetadata.Dispose();
                        
                        if (metadataAreEqual)
                        {
                            IFileManager childFileSetNewMetadata = new FileManager(eventMsg.FileName, eventMsg.Context, needSaveToFile: true, needCreatePublicMarker: true);

                            var success = childFileSetNewMetadata.CreateMetadataForChild(parentMetadata.Metadata.Guid, parentMetadata.Metadata.Marker, out newMetadataDTO);
                            
                            if (success)
                            {
                                _fileDataCache.Add(eventMsg.FileName, newMetadataDTO);
                                eventMsg.ParentGuidDoc = parentMetadata.Metadata.Guid;
                                break;
                            }
                            //Logger.Error($"##### SOMETHING WAS WRONG.");
                            childFileSetNewMetadata.Dispose();
                            // Если зависание останется, удалить.
                            IFileManager childFileOpenForMetadataInit = new FileManager(eventMsg.FileName, eventMsg.Context, needSaveToFile: true, needCreatePublicMarker: true);

                            var tempMetadataInit = childFileOpenForMetadataInit.Metadata;
                            tempMetadataInit = null;
                            childFileOpenForMetadataInit.Dispose();
                        }
                        return;
                    }
                    catch (Exception e)
                    {
	                    //Logger.Error($"{ex.Message}");
                    }
                    if (countOfTryies == 100) break;
                    countOfTryies++;
                    Thread.Sleep(1000);
                }
                // Если длина не одинакова и предпоследние символы у копируемого (N), а у исходного последние символы копия, то пропускаем отправку сообщения.
                if (FilePathsAreCloning(Path.GetFileNameWithoutExtension(eventMsg.CloningFileName), Path.GetFileNameWithoutExtension(eventMsg.FileName)))
                {
                    return;
                }
            }
            //____________________________________
            FileDataCacheElement fileCache = new FileDataCacheElement();
            // если документ переименовали, (или перетащили?)то переименуем перед отправкой.
            if (eventMsg.DocEvent.Equals(eEventType.DocRenamed) || eventMsg.DocEvent.Equals(eEventType.DocMoved))
            {
                if (_fileDataCache.Remove(fileCache.FileName))
                {
                //Logger.Trace(...);
                }
                eventMsg.FileName = eventMsg.CloningFileName;
                _fileDataCache.Add(fileCache.FileName, fileCache.Metadata);
            }

            // Отправить сообщение
            bool res = EventSender.TrySendEvent(_clientApiManager, eventMsg);
            if (!res)
            {
                //Logger.Error(...);
            }
               //Logger.Trace(...);
        }
```

Как видно для каждого события нужно сделать разные никак не связанные с отправкой на сервер действия. Удручает тот факт что события открытия файла локают сам файл, а он еще может быть не закрыт в другом главном потоке.
Как вариант сделать класс обработчик, который будет реализовывать 1 метод(ad-hoc полиморфизм)  в зависимости от типа события. однако это пока не мой уровень и городить множество классов в рабочий проект на данном этапе - опасная затея.
На данный момент буду лишь дробить в приватные методы, ведь метод подготовки к отправке сообщения имеет место быть в методе самого отправке сообщения, как метод резать хлеб в методе приготовить бутерброд.

Получается что здесь применяется декомпозиция из за чрезмерности операций никак не стыкующихся со смыслом метода.
# 2 Вызов метаданных документа с внешней логикой
Есть класс ManagerOfFile благодаря которому мы можем узнать метаданные документа. В конструкторе мы указываем булевы, нужно ли нам создать метаданные или нет. Метадату можем получить только вызвав get свойство публичного поля метадата, это единственный способ, ибо только он указан в интерфейсе 
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
    }
```
```cs
	/// <summary>
    /// Управление файлом.
    /// </summary>
    public class ManagerOfFile : IManagerOfFile
    {
        bool _isReadOnly = false; 
        /// <summary>
        /// Путь до файла.
        /// </summary>
        private string _filePath;
        /// <summary>
        /// Флаг необходимости сохранить данные из памяти процесса в файл после уничтожения объекта.
        /// </summary>
        private bool _needSaveToFile;
        /// <summary>
        /// Флаг необходимости в создании открытой метки.
        /// </summary>
        private bool _needCreatePublicMarker;
        ...
        /// <summary>
        /// Текущие метаданные в памяти.
        /// </summary>
        public MetadataDto Metadata{get...;set...}
    }
```
Get свойство вызывает ряд операций, сначала проверки, потом открытие, после попытку открыть метадату документа, тут и происходит путаница, ведь если быть невнимательным то можно забыть указать флаг _createmetdata_ отчего он станет false и мы попросту не сможем сгенерировать новую метадату для тех ситуаций когда нам действительно нужно. Как вариант выделить отдельным методом MetadataIsExist при методе Open, а дальше автоматически проставлять нужную нам информацию.

Так же узнал что dispose при using вызывается не всегда, отчего файл открывается но не закрывается и из за этого происходит блок при следующих попытках открытия в разных потоках, и закрывать документ только при dispose может быть ошибкой.

# 3 Генерация маршрута
Вызвав метод Create в контроллере VehicleRouteController происходят кучу операций именно внутри метода, заставляя его быть громоздким
# 3 до
```cs
[HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Create([Bind("VehicleId")] Route route, int Count)
        {
            int countOfCreatedRoutes = 0;
            var vehicleId = route.VehicleId;
            var routeByVehicle = _context.Route.FirstOrDefault(r => r.VehicleId == vehicleId);
            if (routeByVehicle == null)
            {
                var now = DateTime.Now;
                var firstRoute = new Route()
                {
                    VehicleId = route.VehicleId,
                    StartDate = now,
                    IsFinished = false,
                    KilometresTraveled = 0
                };
                _context.Route.Add(firstRoute);
                _context.SaveChanges();

                var startGeopoint = new Geopoint()
                {
                    VehicleId = route.VehicleId,
                    RouteId = firstRoute.Id,
                    point = new NetTopologySuite.Geometries.Point(new NetTopologySuite.Geometries.Coordinate(49.9106 , 8.9484)) { SRID = 4326 },
                    LocationDate = now,

                };
                var endGeopoint = new Geopoint()
                {
                    VehicleId = route.VehicleId,
                    RouteId = firstRoute.Id,
                    point = new NetTopologySuite.Geometries.Point(new NetTopologySuite.Geometries.Coordinate(49.5751 , 8.0942)) { SRID = 4326 },
                    LocationDate = now.AddHours(1),

                };

                _context.Geopoint.Add(startGeopoint);
                _context.Geopoint.Add(endGeopoint);
                _context.SaveChanges();

                firstRoute.LocationStart = startGeopoint.Id;
                firstRoute.LocationEnd = endGeopoint.Id;
                firstRoute.EndDate = endGeopoint.LocationDate;

                _context.Route.Update(firstRoute);
                _context.SaveChanges();

                countOfCreatedRoutes++;
            }


            for (;countOfCreatedRoutes < Count; countOfCreatedRoutes++)
            {
                int KilometresTraveled = 0;
                // получаем последний трек
                var lastRoute = _context.Route.Where(r => r.VehicleId == route.VehicleId).OrderBy(x => x.Id).Last();
                // получаем от него конечную дату и конечный геопоинт   
                var endDateOfLastRoute = lastRoute.EndDate.Value.AddHours(18);
                var endLocationOfLastRoute = _context.Geopoint.FirstOrDefault(g => g.Id == lastRoute.LocationEnd);

                //1. Создать рут и получить его айдишник.
                var newRoute = new Route()
                {
                    VehicleId = route.VehicleId,
                    StartDate = endDateOfLastRoute.AddHours(1),
                    IsFinished = false,
                    KilometresTraveled = 0
                };
                _context.Route.Add(newRoute);
                _context.SaveChanges();

                // Создать стартовую геопоинт, добавить в бд, присвоить ньюруту стартовый айди.
                // генерим 10 геопоинтов
                var firstGeopoint = _openRouteServiceApiClient.CreateRandomCoordinateFromStartXY(endLocationOfLastRoute.point.X,
                    endLocationOfLastRoute.point.Y,
                    MaxDistance);
                KilometresTraveled += firstGeopoint.Item3;
                var geopoint = new Geopoint()
                {
                    VehicleId = route.VehicleId,
                    RouteId = newRoute.Id,
                    point = new NetTopologySuite.Geometries.Point(new NetTopologySuite.Geometries.Coordinate(firstGeopoint.Item1, firstGeopoint.Item2)) { SRID = 4326 },
                    LocationDate = endDateOfLastRoute.AddHours(1),

                };
                _context.Geopoint.Add(geopoint);
                _context.SaveChanges();

                newRoute.LocationStart = geopoint.Id;
                newRoute.StartDate = geopoint.LocationDate;

                double bufferX = geopoint.point.X;
                double bufferY = geopoint.point.Y;
                var bufferDate = geopoint.LocationDate;
                for (int i = 0; i < 10; i++)
                {
                    firstGeopoint = _openRouteServiceApiClient.CreateRandomCoordinateFromStartXY(
                        bufferX,
                        bufferY,
                        MaxDistance);
                    KilometresTraveled += firstGeopoint.Item3;
                    geopoint = new Geopoint()
                    {
                        VehicleId = route.VehicleId,
                        RouteId = newRoute.Id,
                        point = new NetTopologySuite.Geometries.Point(new NetTopologySuite.Geometries.Coordinate(firstGeopoint.Item1, firstGeopoint.Item2)) { SRID = 4326 },
                        LocationDate = bufferDate.AddHours(1)
                    };

                    _context.Geopoint.Add(geopoint);
                    _context.SaveChanges();

                    bufferX = geopoint.point.X;
                    bufferY = geopoint.point.Y;
                }
                firstGeopoint = _openRouteServiceApiClient.CreateRandomCoordinateFromStartXY(
                         bufferX,
                         bufferY,
                         MaxDistance);
                KilometresTraveled += firstGeopoint.Item3;
                geopoint = new Geopoint()
                {
                    VehicleId = route.VehicleId,
                    RouteId = newRoute.Id,
                    point = new NetTopologySuite.Geometries.Point(new NetTopologySuite.Geometries.Coordinate(firstGeopoint.Item1, firstGeopoint.Item2)) { SRID = 4326 },
                    LocationDate = bufferDate.AddHours(1)
                };
                _context.Geopoint.Add(geopoint);
                _context.SaveChanges();
                // конечная дата равна дате последнего получившегося геопоинта
                newRoute.EndDate = geopoint.LocationDate;
                // isfinished = true;
                newRoute.IsFinished = true;
                newRoute.KilometresTraveled = KilometresTraveled;
                newRoute.LocationEnd = geopoint.Id;

                _context.Route.Update(newRoute);
                _context.SaveChanges();
            }
            return View();
        }
```
Декомпозируем и разобьем его, делая более читаемым
# 3 после

```cs
// POST: VehicleRoute/Create
        // To protect from overposting attacks, enable the specific properties you want to bind to.
        // For more details, see http://go.microsoft.com/fwlink/?LinkId=317598.
        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Create([Bind("VehicleId")] Route route, int Count)
        {
            int countOfCreatedRoutes = 0;
            var vehicleId = route.VehicleId;
            var routeByVehicle = _context.Route.FirstOrDefault(r => r.VehicleId == vehicleId);
            if (routeByVehicle == null)
            {
                countOfCreatedRoutes = InitializeNewRoute(route, countOfCreatedRoutes);
            }

            countOfCreatedRoutes = GenerateNewRoutes(route, Count, countOfCreatedRoutes);
            return View();
        }
```

Метод стал удобочитаемым и более понятным, если произойдет баг во время генерации маршрута, то мы быстрее сможем соориентироваться, где именно он может находиться.

Пытался найти другие примеры но не смог, и непонятно толи плохо искал, толи я стал лучше писать код и потому уже вижу на какие действия можно разделить сразу же.
