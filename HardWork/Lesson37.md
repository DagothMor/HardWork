# 2 
2 примера обучающих, 1 рабочий
## 2.1 Vehicle

От Landvehicle можно избавиться, Он несет в себе лишь обобщение.
```cs
abstract class Vehicle
{
    public string Type { get; set; }
    public int MaxSpeed { get; set; }
    public int SeatCount { get; set; }

    public abstract void Move();
}
abstract class Landvehicle : Vehicle
{

}

class Car : Landvehicle
{
    public Car()

    public override void Move()
}

class Plane : Vehicle
{
    public Plane()

    public override void Move()

    public void Fly()
}

class Ship : Vehicle
{
    public Ship()

    public override void Move()
}

class Bicycle : Vehicle
{
    public Bicycle()

    public override void Move()
}


```

# 2.2

Так же можно избавиться от обобщающих классов Grocery, Entertaining, DeliveryOfProperty
```cs
abstract class Business
{
    public string Name { get; set; }
    public string Address { get; set; }
    public decimal Revenue { get; set; }
    public int EmployeeCount { get; set; }

    public abstract void ServeCustomer();

    public virtual void PrintInfo()
    {
    }
}

class Grocery : Business
{

}

class Entertaining : Business
{

}

class DeliveryOfProperty : Business
{

}

class Restaurant : Grocery
{
    public override void ServeCustomer()
    {
    }

    public override void PrintInfo()
    {
        base.PrintInfo();
        Console.WriteLine("Cuisine: Italian");
    }
}

class Shop : Grocery
{
    public override void ServeCustomer()
    {
    }
}

class Hotel : DeliveryOfProperty
{
    public override void ServeCustomer()
    {
    }
}

class Cinema : Entertaining
{
    public override void ServeCustomer()
    {
    }
}

```
# 3

# 3.1
Логично, что все самолеты перед тем как начать лететь, набирают скорость на полосе "двигаясь".
Выделим каждое действие в обязанность летать, ездить, плавать.
```cs
interface ITransport
{
    string Type { get; set; }
    int MaxSpeed { get; set; }
    int SeatCount { get; set; }
}

interface IMove
{
    void Move();
}

interface IFly
{
    void Fly();
}

interface IFloat
{
    void Float();
}

class Car : ITransport, IMove
{
    public string Type { get; set; } = "Car";
    public int MaxSpeed { get; set; } = 200;
    public int SeatCount { get; set; } = 5;

    public void Move()
}

class Plane : ITransport, IMove, IFly
{
    public string Type { get; set; } = "Plane";
    public int MaxSpeed { get; set; } = 900;
    public int SeatCount { get; set; } = 200;

    public void Move()

    public void Fly()
}

class Ship : ITransport, IMove,IFloat
{
    public string Type { get; set; } = "Ship";
    public int MaxSpeed { get; set; } = 50;
    public int SeatCount { get; set; } = 1000;

    public void Float()
}

class Bicycle : ITransport, IMove
{
    public string Type { get; set; } = "Bicycle";
    public int MaxSpeed { get; set; } = 30;
    public int SeatCount { get; set; } = 1;

    public void Move()
}

```
# 3.2
Бизнес это не только про обслуживание посетителей, это может быть просто производство продукта и продажа агентам.

```cs
interface IBusiness
{
    string Name { get; set; }
    string Address { get; set; }
    decimal Revenue { get; set; }
    int EmployeeCount { get; set; }
}

interface IServeCustomer
{
    void ServeCustomer();
}
interface IProduceProduct
{
    void ProduceProduct();
}

class Restaurant : IBusiness, IServeCustomer
{
    public string Name { get; set; }
    public string Address { get; set; }
    public decimal Revenue { get; set; }
    public int EmployeeCount { get; set; }

    public void ServeCustomer()
}

class Shop : IBusiness, IServeCustomer
{
    public string Name { get; set; }
    public string Address { get; set; }
    public decimal Revenue { get; set; }
    public int EmployeeCount { get; set; }

    public void ServeCustomer()
}

class Hotel : IBusiness, IServeCustomer
class Cinema : IBusiness, IServeCustomer

class TextileFactory : IBusiness, IProduceProduct
{
    public string Name { get; set; }
    public string Address { get; set; }
    public decimal Revenue { get; set; }
    public int EmployeeCount { get; set; }

    public void ProduceProduct()
}

```
# 3.3
в зависимости от функциональности нашего приложения, мы можем через интерфейсы управлять возможностями документов старого формата, и нового
```cs
interface IDocument
{
    string Name { get; set; }
    DateTime CreationDate { get; set; }
}

interface ICRUD
{
    void Create();
    void Read();
    void Update();
    void Delete();
}

interface IContentExtractable
{
    string GetText();
    Image GetImage();
}

class BinaryDoc : IDocument, ICRUD
{
    public string Name { get; set; }
    public DateTime CreationDate { get; set; }

    // crud
}

class XDoc : IDocument, ICRUD, IContentExtractable
{
    public string Name { get; set; }
    public DateTime CreationDate { get; set; }

    // crud

    public string GetText()
    {
        return "XML document text";
    }

    public Image GetImage()
    {
        return new Bitmap("xml_document.bmp");
    }
}

```

# Итог
Вариант с избавлением от промежуточного обобщающего класса безболезненно может показаться разумным, НО.
Нужно отталкиваться от абстракций в рамках бизнес требования(задачи, спецификаций).
На момент разработки, обобщающий класс документ-договор может и быть легко удален, но что если в будущем нам в рамках ЭДО нужно работать с документами не как с сущностями несущими в себе функционал Iподписываемый,Iэцп... , а как непосредственно с классами. 

Для меня интерфейс это контракт, дал клятву Гиппократа - будь добр выполняй метод лечить кого угодно, теперь ты лекарь. Но не доктор, не врач, а просто лекарь. мы работает с объектом не как с врачом, а как с просто объектом, который реализует Nый функционал.
Нужно явно разделять Абстракции и сущности с контрактами(обязанностями), перевод из одного в другое и наоборот имеет смысл только если мы грамотно составим спецификацию. Возможно, мы жестко привязываем функционал, чтобы в будущем при разработке лающие(от поведения собаки) кошки, не могли есть собачий корм. Какова вероятность в разработке, что случится с женщиной, если она весит как утка, а значит сделана из дерева(дерево плавает как и утка), а значит горит в огне, а что еще горит в огне? 

Правильно.
Ведьма.