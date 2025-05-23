# 1 Конвертация в отдельную сущность

Наша Entity документа состоит из json, у нее есть список аттрибутов, в котором есть поле Group, являющийся массивом объектов
В рамках интеграций мы

1.1 получаем сущность
1.2 получаем массив group
1.3 преобразовываем в запрос во внешнюю интеграцию, поле одно из которых булева, значение которой меняется в зависимости от бизнес правила суммы всех элементов Group 
1.4 Вычленение нужных нам элементов group
1.5 Работа с ними после получения ответа от внешней интеграции

Код сгенерирован, это условный пример наших действий
```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text.Json;

public class GroupItem
{
    public int Id { get; set; }
    public int Value { get; set; }
}

public class Entity
{
    public Dictionary<string, object> Attributes { get; set; }
}

public class IntegrationRequest
{
    public bool IsSumGreaterThanThreshold { get; set; }
    public List<GroupItem> FilteredGroup { get; set; }
}

public class IntegrationResponse
{
    public List<GroupItem> UpdatedGroup { get; set; }
}

public class JsonProcessor
{
    public static IntegrationRequest ProcessEntity(string json, int threshold)
    {
        // Десериализация JSON в объект Entity
        var entity = JsonSerializer.Deserialize<Entity>(json);

        // Извлечение массива Group
        var group = JsonSerializer.Deserialize<List<GroupItem>>(entity.Attributes["Group"].ToString());

        // Вычисление суммы всех элементов Group
        int sum = group.Sum(item => item.Value);

        // Определение булевого значения на основе бизнес-правила
        bool isSumGreaterThanThreshold = sum > threshold;

        // Вычленение нужных элементов Group (например, только те, у которых Value > 15)
        var filteredGroup = group.Where(item => item.Value > 15).ToList();

        // Создание запроса для внешней интеграции
        var integrationRequest = new IntegrationRequest
        {
            IsSumGreaterThanThreshold = isSumGreaterThanThreshold,
            FilteredGroup = filteredGroup
        };

        return integrationRequest;
    }

    public static Entity UpdateEntity(Entity entity, string responseJson)
    {
        // Десериализация ответа от внешней интеграции
        var response = JsonSerializer.Deserialize<IntegrationResponse>(responseJson);

        // Обновление массива Group в сущности на основе ответа
        entity.Attributes["Group"] = response.UpdatedGroup;

        return entity;
    }

    public static void Main()
    {
        string json = @"{
            ""Attributes"": {
                ""Group"": [
                    {""Id"": 1, ""Value"": 10},
                    {""Id"": 2, ""Value"": 20},
                    {""Id"": 3, ""Value"": 30}
                ]
            }
        }";

        int threshold = 50;
        var integrationRequest = ProcessEntity(json, threshold);

        Console.WriteLine($"IsSumGreaterThanThreshold: {integrationRequest.IsSumGreaterThanThreshold}");
        Console.WriteLine("FilteredGroup:");
        foreach (var item in integrationRequest.FilteredGroup)
        {
            Console.WriteLine($"Id: {item.Id}, Value: {item.Value}");
        }

        // Предположим, что мы получили ответ от внешней интеграции
        string responseJson = @"{
            ""UpdatedGroup"": [
                {""Id"": 1, ""Value"": 15},
                {""Id"": 2, ""Value"": 25},
                {""Id"": 3, ""Value"": 35}
            ]
        }";

        // Десериализация исходного JSON в объект Entity
        var entity = JsonSerializer.Deserialize<Entity>(json);

        // Обновление сущности на основе ответа
        var updatedEntity = UpdateEntity(entity, responseJson);

        // Сериализация обновленной сущности обратно в JSON
        string updatedJson = JsonSerializer.Serialize(updatedEntity);
        Console.WriteLine(updatedJson);
    }
}

```

Других примеров за последние месяцы работы я не найду, на ум приходит только сериализация и десериализация объекта с доп бизнес логикой, но не уверен что это прям поднятие и понижение уровня абстракций.
# Итог
Вспомнился старый китайский фильм два война, и легендарная фраза, давшая персонажу Джет Ли, новую жизнь - "Оставь свою ношу"

Так и мы, беря сущность извне, убираем все лишнее, оставляя нужную нам суть.
