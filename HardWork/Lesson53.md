Думаю что CPS применяется у меня в одном маленьком и непримечательном пэт проекте, а именно конвейер шагов.

```cs
protected override Task InitializeProcessAsync(|||||| firstStep, CancellationToken cancellationToken)
{
    firstStep
        .SendByTransparentProxy<Guid, ||||Request, ||||Response>(
            returnStepId: ||||,
            configureAction: ||||)
        .AddNextStep<ProcessAuthTokenStep>(
            executeCondition: |||| => ||||?.Count > 0)
        .AddNextStep<LoadEntityDataLightStep>()

        .SendCommandsByTransparentProxy<||||, ||||Request, ||||Response>(
            returnStepId: ||||StepId,
            configureAction: ConfigureCommand)
        .AddNextStep<||||Step>(
            executeCondition: response => response?.Count > 0)

            .AddNextSubStep<||||Step<IReadOnlyCollection<||||Model>?>>(
                executeCondition: response => response.First().IsNo||||Purchase && !response.First().AppNoPriceOffer,
                configureNextStep: Processing||||Protocol)
            .AddNextSubStep<||||Step<IReadOnlyCollection<||||Model>?>>(
                executeCondition: response => response.First().IsNo||||Purchase && response.First().AppNoPriceOffer,
                configureNextStep: ProcessingWithNo||||ProtocolWithComment)
            .AddNextSubStep<||||Step<IReadOnlyCollection<||||Model>?>>(
                executeCondition: response => !response.First().IsNo||||Purchase,
                configureNextStep: ProcessingWith||||||Protocol);

    return Task.CompletedTask;
}

```
Благодаря архитектуре мы можем создавать классы шагов(указывая что должно приходить на вход от предыдущего шага и то что должно вернуться) и вызывать их логику, последовательно, с возможностью восстановления.
