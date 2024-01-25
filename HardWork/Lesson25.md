Теперь понимаю почему все стараются избегать в .NET EntityFrameworkCore, в угоду DAPPER или ADO.NET

Вот например существует вероятность что для простого get мы забыли добавить важную функцию AsNoTracking

https://metanit.com/sharp/entityframework/4.8.php

```
Чтобы данные не помещались в кэш, применяется метод AsNoTracking(). При его применении возвращаемые из запроса данные не кэшируются. А это означает, что Entity Framework не производит какую-то дополнительную обработку и не выделяет дополнительное место для хранения извлеченных из БД объектов.
```

Это конечно весомый минус, однако более показательный это генерация запроса из большого вложенного Linq запроса, показательно будет, вызвать его, запустив SQL Profiler для ssms, ну или log_statement config setting для postgres
https://www.postgresql.org/docs/current/runtime-config-logging.html#guc-log-statement
# 1 
До
```csharp
var driverEnterprises = _managerRepository
                    .GetAll(includeProperties: "Enterprises")
                    .Where(e => e.Id == user.Id)
                    .SelectMany(e => e.Enterprises)
                    .Select(x => new Enterprise()
                    {
                        Id = x.Id,
                        Name = x.Name,
                        BusinessType = x.BusinessType,
                        Address = x.Address,
                        CEO = x.CEO,
                        ContactInfo = x.ContactInfo,
                        EmployeeCount = x.EmployeeCount,
                    })
                    .ToList();
```

должно перехватится так
```sql
SELECT e.Id, e.Name, e.BusinessType, e.Address, e.CEO, e.ContactInfo, e.EmployeeCount 
FROM Managers AS m 
INNER JOIN ManagerEnterprises AS me 
ON m.Id = me.ManagerId 
INNER JOIN Enterprises AS e ON me.EnterpriseId = e.Id 
WHERE m.Id = @userId
```
Inner join мощный, однако из за возможного множества таблиц или сложных условий, запрос может стать ресурсоемким

после:

```csharp
string query = @"SELECT e.Id, e.Name, e.BusinessType, e.Address, e.CEO, e.ContactInfo, e.EmployeeCount
                 FROM Enterprises AS e
                 JOIN ManagerEnterprises AS me ON e.Id = me.EnterpriseId
                 WHERE me.ManagerId = @ManagerId;";
var driverEnterprises = connection.Query<Enterprise>(query, new { ManagerId = user.Id }).ToList();
```
# 2 
До
```cs
var applicationDbContext = _orderRepository
                        .GetAll(includeProperties: "Vehicle")
                        .Where(u => u.EnterpriseId == enterprise.Id)
                        .Where(o => o.ActivityStatus == WebConst.IsActive);
```

Возможен такой исход потому что EFC отталкивается от контекста(Отношения,жадная ленивая загрузки,навигационные свойства...)
```sql
SELECT o.* 
FROM Orders AS o
LEFT JOIN Vehicles AS v
ON o.VehicleId = v.Id 
WHERE o.EnterpriseId = @enterpriseId AND o.ActivityStatus = @activityStatus;
```

и если маленький проект умещается в голове и это можно как то держать, то в больших - выделки не стоит.

после
``` cs
string query = @"SELECT o.*, v.* 
FROM Orders AS o 
LEFT JOIN Vehicles AS v 
ON o.VehicleId = v.Id 
WHERE o.EnterpriseId = @EnterpriseId AND o.ActivityStatus = @ActivityStatus;";
```

Запрос не изменился, но при будущем нарастании проекта, будут увеличиваться количество отношений, что может сыграть само по себе злую шутку. Нужно еще учесть что Dapper сам по себе не отслеживает изменения, удаление обновление вставка, все ручками.

# 3
Попробуем получить геопозиции связанные с маршрутами по отрезку времени.
до
```cs
            var geopoints = _context.Route
                .Where(r => r.VehicleId == ownerParameters.VehicleId
                && r.StartDate >= routeStartDate && r.StartDate <= routeEndDate
                && r.EndDate >= routeStartDate
                && r.EndDate <= routeEndDate)
                .SelectMany(r => r.Geopoints).AsNoTracking().ToList();
```
Возможный вариант исполнения
sql
```sql
SELECT g.* 
FROM Routes AS r 
INNER JOIN Geopoints AS g 
ON r.Id = g.RouteId 
WHERE r.VehicleId = @VehicleId 
AND r.StartDate >= @RouteStartDate 
AND r.StartDate <= @RouteEndDate 
AND r.EndDate >= @RouteStartDate 
AND r.EndDate <= @RouteEndDate;
```

После
```sql
string query = @"SELECT g.* 
	FROM Geopoints AS g 
			INNER JOIN Routes AS r 
			ON g.RouteId = r.Id 
			WHERE r.VehicleId = @VehicleId AND r.StartDate 
				BETWEEN @RouteStartDate AND @RouteEndDate AND r.EndDate 
				BETWEEN @RouteStartDate AND @RouteEndDate;"; 
var geopoints = connection.Query<Geopoint>(query, new { VehicleId = ownerParameters.VehicleId, RouteStartDate = routeStartDate, RouteEndDate = routeEndDate }).ToList();
```

Конечно разница между and и between судя по изученному мала, однако здесь играет роль удобочитаемость.

Итог

Если Dapper лучше по многим параметрам, то почему к нему все не перейдут от EFC?
Возможно из за модульности, скорости разработки, судя по последнему dotnext он активно развивается но все еще не поспевает за даппером.
Контекст, все зависит от контекста, однако слова "нужно быстро сделать и выложить" работают на меня все меньше и меньше, будто хочется проигнорировать и продолжать тренироваться, делать правильно но быстрее нежели быстро но неправильно в будущем.