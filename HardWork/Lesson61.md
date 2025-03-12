# 1 filter
После получения списка аттрибутов я проходился по каждому элементу и проверял валидность определенных значений, был выделен метод и сам проход заменен на filter, благодаря чему читабельность увеличилась в разы 

```cs
var unfilledPropertyNames = new StringBuilder();

var invalidGroupPozitions = groupPozitions
    .Select((gp, i) => new { GroupPozition = gp, Index = i })
    .Where(item =>
        string.IsNullOrEmpty(item.GroupPozition.ProjectStage.First().ProjectStageCode) ||
        string.IsNullOrEmpty(item.GroupPozition.FinancialResponsibilityCenter.First().FinancialResponsibilityCenterCode) ||
        (item.GroupPozition.BudgetCheckStart.HasValue && item.GroupPozition.BudgetCheckStart.Value.Year == DateTime.MinValue.Year) ||
        (item.GroupPozition.BudgetCheckFin.HasValue && item.GroupPozition.BudgetCheckFin.Value.Year == DateTime.MinValue.Year) ||
        !item.GroupPozition.Znp_ExpectedDate.HasValue ||
        (string.IsNullOrEmpty(item.GroupPozition.GrExpenditureProductGroup?.First().Code) &&
         string.IsNullOrEmpty(item.GroupPozition.BudgetItemName?.First().BudgetItem)))
    .ToList();

```

# 2 reduce

На основе списка аттрибутов, нужно было создать кастомный словарь, разумеется происходила его инициализация и заполнение через цикл.

reduce не существует в c#, но есть aggregate

```cs
var productGroupDictionary = groupPozitions
    .Where(gp => !string.IsNullOrEmpty(gp.GrExpenditureProductGroup?.FirstOrDefault()?.Code))
    .Aggregate(
        new Dictionary<ProductGroup, float>(),
        (acc, gp) =>
        {
            var projectStageCode = gp.ProjectStage.FirstOrDefault().ProjectStageCode;
            var financialResponsibilityCenterCode = gp.FinancialResponsibilityCenter.FirstOrDefault().FinancialResponsibilityCenterCode;
            var expectedDate = gp.Znp_ExpectedDate ?? DateTime.MinValue;
            var productGroup = new ProductGroup(
                projectStageCode,
                financialResponsibilityCenterCode,
                gp.GrExpenditureProductGroup?.FirstOrDefault()?.Code,
                expectedDate.Year,
                expectedDate.Month);

            if (acc.ContainsKey(productGroup))
            {
                acc[productGroup] += gp.SumExpenditureGroup.Value;
            }
            else
            {
                acc[productGroup] = gp.SumExpenditureGroup.Value;
            }

            return acc;
        });

```

# 3 В продолжении невалидных элементов аттрибутного состава

```cs
var unfilledPropertyNames = invalidGroupPozitions
    .Aggregate(
        new StringBuilder(),
        (acc, item) =>
        {
            var groupPozition = item.GroupPozition;
            var index = item.Index;

            if (string.IsNullOrEmpty(groupPozition.ProjectStage.First().ProjectStageCode))
            {
                acc.Append($"Ошибка: для {index}-ого group_Position не заполнен ProjectStageCode | ");
            }
            //
			//...
			//

            return acc;
        });

string unfilledPropertyNamesString = unfilledPropertyNames.ToString();

```


# 4 map
Он же аналог select для c#, после инициализации кастомного словаря, проходимся по элементам и указываем значение аттрибута в зависимости от ключа.

```cs
var updatedGroupPozitions = groupPozitions
    .Select((groupPozition, i) =>
    {
        var projectStageCode =//...
        //...
```

# итог
замена цикла действительно выходит красивее и лаконичнее, глаза конечно все равно будут теряться, но не с такой силой, если посмотришь на код впервые.
