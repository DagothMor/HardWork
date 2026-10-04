# 1.1
## До
```cs
_excelReader.ReadColumn(fileData,2,1)
```
что 2 1?
## После
```cs
_excelReader.ReadColumn(fileData,startRow: 2,ColumnNumber:1)
```

# 1.2

## До 
```cs
if(application.Status == 4 && application.DaysOverdue > 30){
	AddToReport(application);
}
```

Не понятно что 4 что 30

# После
```cs
if(application.Status == applicationStatus.Active && application.DaysOverdue > ReportRules.OverdueDaysthreshold){
	AddToReport(application);
}
```

# 2
Реализация выглядит тяжелее чем само правило

# 2.1
```cs
if(report.isReady)
{
	if(user.CanDownload)
		return true;
		else return false;
}

return false;
```

после

```cs
return report.IsReady && user.CanDownload;
```

# 2.2

```cs
if(payments.Count == 0)
	return 0m;
	
decimal total = 0m;

foreach (var payment in payments)
{
	total += payment.Amount;
}

return total;
```

Хотя можно банально

```cs
return payments.Sum(payment => payment.Amount);
```
