К сожалению на рабочем проекте не было примеров, где можно было бы использовать этот прием, рассмотрим  этот пример:
```cs
public async Task<IReadOnlyCollection<EntityDataLightViewModel>> GetLinkedEntitiesAsync(
    EntityLinkViewModel[] entityLinkViewModels,
    Func<EntityLinkViewModel, bool> filter,
    int loadChuckSize,
    CancellationToken cancellationToken)
{
    var loadedLinkDocuments = new List<EntityDataLightViewModel>();
    var entityLinkIds = entityLinkViewModels
        .Where(filter)
        .Select(x => x.Reference);

    foreach (var entityLinkIdChunk in entityLinkIds.Chunk(loadChuckSize))
    {
        var loadLinkDocumentTasks =
            entityLinkIdChunk.Select(entityLinkId =>
                GetEntityDataWithSchemaAsync(entityLinkId, cancellationToken));

        var loadedLinkDocument = await Task.WhenAll(loadLinkDocumentTasks);
        loadedLinkDocuments.AddRange(loadedLinkDocument);
    }

    return loadedLinkDocuments;
}
```

Проблема в том, что мы получаем сущности из внутреннего апи, и мы по факту зависим от него, палок в колеса вставляет не меньше чем внешние контуры.
да, мы можем сделать множество методов, где каждый будет отвечать за получение связанных сущностей по определенному типу сущности, но их достаточно много, и описание каждой сущности + сериализация аттрибутивного состава(который очень любит меняться) создаст долгоиграющую историю с постоянным стучанием к коллегам другого цеха.
Так и работаем.
