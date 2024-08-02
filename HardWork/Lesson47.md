
Устроившись на новую работу мне все страшнее брать примеры из кода, потому делаю максимально размыто.
# 1

## Было
Частые проверки на null.
```cs
public void SendContract(ProxyController proxyController, Contract contract)
{
    if (proxyController != null)
    {
        proxyController.SendContractToNDA(contract);
    }
}

```

## Стало
Реализуем наследника, который будет переопределять все методы родителя, в логику можем записать или вызов исключений, или оповещение через логгер.
```cs
// Если посредник отключен, или неправильно подхватились конфигурации.
public class BadProxyController : ProxyController
{
    public override void SendContractToNDA(Contract contract)
    {
	    // Logger.Fatal...
    }
}

public void SendContract(ProxyController proxyController, Contract contract)
{
    proxyController.SendContractToNDA(contract);
}

```
Использовался паттерн Null Object
# 2

## Было
```cs
public void GetContracts(Trade trade)
{
    if (trade.Type == TradeType.Open)
    {
        GetOpenContracts(trade);
    }
    else if (trade.Type == TradeType.Closed)
    {
        GetClosedContracts(trade);
    }
}

```
## Стало

```cs
public abstract class Trade
{
    public abstract void GetContracts();
}

public class OpenTrade : Trade
{
    public override void GetContracts()
    {
        // Логика получения контрактов из открытых торгов
    }
}

public class ClosedTrade : Trade
{
    public override void GetContracts()
    {
	    // Проверка цифровой подписи
        // Логика получения контрактов из закрытых торгов
    }
}

public void GetContracts(Trade trade)
{
    trade.GetContracts();
}

```

Использовался прием полиморфизма для обработки различных типов торгов.

# 3

## Было
```cs
public void ProcessDocument(Document document)
{
    if (document.Type == DocumentType.Contract)
    {
        ProcessContract(document);
    }
    else if (document.Type == DocumentType.Addendum)
    {
        ProcessAddendum(document);
    }
}

```
## Стало

```cs
public abstract class Document
{
    // Общие свойства и методы для всех документов
}

public class Contract : Document
{
    public void Process()
    {
        // Логика обработки договора
        Console.WriteLine("Processing Contract");
    }
}

public class Addendum : Document
{
    public void Process()
    {
        // Логика обработки доп соглашения
        Console.WriteLine("Processing Addendum");
    }
}

public class DocumentProcessor
{
    public void ProcessDocument(Document document)
    {
        switch (document)
        {
            case Contract contract:
                contract.Process();
                break;
            case Addendum addendum:
                addendum.Process();
                break;
            default:
                throw new ArgumentException("Unknown document type");
        }
    }
}
```
Прием - костыльный на шарпе pattern matching.

# 4 
Документы в формате xml, и поля у этого документа могут меняться в зависимости от хотелок внешнего поставщика.

# Было
```cs
public void ValidateForm(Dictionary<string, object> data)
{
    if (!data.ContainsKey("name") || string.IsNullOrEmpty(data["name"] as string))
    {
        throw new ArgumentException("Name is required");
    }
    if (!data.ContainsKey("email") || string.IsNullOrEmpty(data["email"] as string))
    {
        throw new ArgumentException("Email is required");
    }
    if (!data.ContainsKey("age") || (int)data["age"] < 18)
    {
        throw new ArgumentException("Age must be at least 18");
    }
}

```
# Стало
Простроим абстракцию валидатора xml документа.
```cs
using System;
using System.Xml;

public abstract class XmlFieldValidator
{
    public abstract void Validate(XmlDocument document);
}

public class XmlPresenceValidator : XmlFieldValidator
{
    private readonly string _fieldName;

    public XmlPresenceValidator(string fieldName)
    {
        _fieldName = fieldName;
    }

    public override void Validate(XmlDocument document)
    {
        var node = document.SelectSingleNode($"//{_fieldName}");
        if (node == null || string.IsNullOrEmpty(node.InnerText))
        {
            throw new ArgumentException($"{_fieldName} is required");
        }
    }
}

public class XmlAgeValidator : XmlFieldValidator
{
    public override void Validate(XmlDocument document)
    {
        var node = document.SelectSingleNode("//age");
        if (node == null || !int.TryParse(node.InnerText, out int age) || age < 18)
        {
            throw new ArgumentException("Age must be at least 18");
        }
    }
}

public class XmlFormValidator
{
    private readonly XmlDocument _document;
    private readonly XmlFieldValidator[] _validators;

    public XmlFormValidator(XmlDocument document, params XmlFieldValidator[] validators)
    {
        _document = document;
        _validators = validators;
        Validate();
    }

    private void Validate()
    {
        foreach (var validator in _validators)
        {
            validator.Validate(_document);
        }
    }
}

```
XmlPresenceValidator - проверяет наличие и не пустоту определенного поля
XmlAgeValidator - проверяет уже конкретное значение.

сам XmlFormValidator принимает документ для валидации и массив валидирующих параметров, можно кидать синглтоном через DI

Остальные примеры очень часто пересекаются с 1 2 3 пунктами.
# Итог
Избавиться от ifов можно, по большей части благодаря типам и состояниям, грамотно простроив типы можно снизить явно цикломатическую сложность.