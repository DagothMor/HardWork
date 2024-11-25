# 3
Сущность шага представляет собой атомарную единицу логики, которая должна выполниться\выбросить исключение.
```cs
/// <summary>
/// Абстрактный базовый класс шага БП
/// </summary>
/// <typeparam name="TInput">Тип входных данных</typeparam>
/// <typeparam name="TOutput">Тип исходящих данных</typeparam>
public abstract class BpStepGeneric<TInput, TOutput> : IBpStepGeneric<TInput, TOutput>
```

Мы можем создать унифицированный шаг, который обязан выполнить базовую инициализацию
```cs
public class InitStep : BpStepGeneric<Guid, Guid>
```
Или же конкретную логику, например отправить запрос во внешний контур
```cs
SendRequestStep : BpStepGeneric<IReadOnlyCollection<EntityDataLightViewModel>, string>
```

___
Сущность бизнес процесса представляет собой пайплайн шагов, мы выстраиваем последовательность шагов внутри него.
```cs
/// <summary>
/// Базовый абстрактный БП
/// </summary>
/// <typeparam name="TFirstStep">Тип первого шага</typeparam>
/// <typeparam name="TInput">Тип входных данных первого шага</typeparam>
public abstract class BProcessBase<TFirstStep, TInput> : IBProcess<TInput>
    where TFirstStep : class, IBpStepWithInput<TInput>
```

Наследуясь от него, мы прописываем последовательность шагов для определенного сценария
```cs
public class GetCurStatProcess : BProcessBase<InitStep, Guid>

```
___
Плагин представляет собой оболочку над процессом, в нем иницилизируются настройки,DI,шедулеры итд
```cs
public abstract class AIntegrationPlugin<TSettings> : IIntegrationPlugin, IIntegrationSyncConfig
    where TSettings: class, new()
```
В примере плагин все еще абстрактный, мы можем сделать наследование от него и получить все необходимые нам настройки и сервисы для плагина с конкретным сценарием(обновление сущностей договоров авиаперелетов, регистрация документов итд итп)
```cs
public abstract class AviaSalesAllInOnePlugin<TSettings> : AIntegrationPlugin<TSettings>
    where TSettings: class, new()
```

# 2

Пример тестирования новых шагов, пайплайна используя FluentAssertions

```cs
[Fact]
public async Task Execute_Test()
{
    PipelineFixture fixture = new PipelineFixture();

    BusinessProcessPipeline pipeline = fixture.CreatePipeline(builder =>
        builder
            .AddFirstStep<FirstTestStep>()
            .AddNextStep<SecondTestStep>()
            .AddNextSubStep<ParentTestStep>(x => x
                .AddNextStep<ChildTestStepA>()
                .AddNextStep<ChildTestStepB>()
            )
            .AddNextStep<LastTestStep>());

    Guid firstStepInput = Guid.NewGuid();

    await pipeline.Execute(firstStepInput);

    var context = fixture.Get<TestStepsContext>();

    context.FirstTestStep.Input.Should().Be(firstStepInput);
    context.SecondTestStep.Input.Should().Be(context.FirstTestStep.Output);
    context.ParentTestStep.Input.Should().Be(context.SecondTestStep.Output);
    context.ChildTestStepA.Input.Should().Be(context.ParentTestStep.Output);
    context.ChildTestStepB.Input.Should().Be(context.ChildTestStepA.Output);
    context.LastTestStep.Input.Should().Be(context.ParentTestStep.Output);
    context.LastTestStep.Output.Should().Be(LastTestStep.Result);

    context.ExecutedStepCount.Should().Be(6);
}
```

# Вывод
Мы явно используем полиморфизма подтипов, заранее построив абстракции мы создаем детей только для конкретных случаев. 
