# 1 безопасный json десериализатор

```
которая делает только одну вещь и делает её хорошо.
```

Функция получения гуида делала плохо, на вход был json element, в котором была лишь попытка парсинга из текста, в сущности из ядра ктото умудрился засунуть массив с 1 элементом :)

была добавлена рекурсивная логика получения первого элемента из массива(кто знает может там будет массив массивов)

```cs
private Guid ParseGuid(in JsonElement property)
{
    if (property.ValueKind == JsonValueKind.Array && TryGetFirstElement(in property, out JsonElement firstElement))
    {
        return ParseGuid(in firstElement);
    }

    if (property.TryGetGuid(out Guid value))
    {
        return value;
    }
	///...
    AddError(in property, "Guid");
    return Guid.Empty;
}
```
# 2 Шаг авторизации

Был перереботан сценарий получения токена авторизации, а именно в классе расширений, был добавлен шаг, состоящий из подшагов:
проверки настроек
генерация request
отправка в очередь
обработка ответа
```cs
public static IBuildProcMain<TResp> SendByTransparentProxy<TIn, TReq, TResponse>(
        this IBuildingProcessMaintainer<TIn> maintainer,
        ConfigureStepDelegate<TIn, TransparentProxyEventBusCommand>? configureAction,
        StepExecuteConditionDelegate<TIn>? executeCondition = null,
        bool writeResponseToAnalystLog = true)
    where TReq : class, ITrPrxyRequest<TInput>
```
