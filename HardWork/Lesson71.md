Все примеры довольно однотипные, опишу саму суть:
Напомню что у нас есть 3 базовых сущности, а именно Плагин, процесс, и шаги
Плагин вызывает процесс и забирает из ioc нужные для процесса сущности, а так же запускает шедулеры.
Процесс состоит из множества шагов
Каждый шаг принимает в себя какой то тип данных и возвращает что то другое.
благодаря такой архитектуре, нам очень удобно собирать сценарии, а именно:

```cs
public static test(T inputData)
{
	pipe.AddFirstStep<StartStep>(inputData)
		.AddProxyRequest<LoginPayload>(inputData)
		.AddNextStep<ProcessRequestStep>(inputData)
		.AddNextStep<GetInfoFromInnerConturStep>(inputData)
		.AddExtraProxyRequest<SendProxyPayload>(infoData)
		.AddNextStep<ProcessRequestStep>(requestData)
		.if()...
		.While(inputData.Request.body.success != "Success"
			=> subpipe.AddProxyRequest<SendProxyPayload>(infoData)
					.AddNextStep<ProcessRequestStep>(requestData)
		)
		

}
```

От базовых шагов, мы можем наследовать и создавать новые, допустим обработка ответов от Контура, быть может он очень любит кидать xml в виде ответа, и потому у нас уже есть переопределенный метод обработки ответа на запрос.
