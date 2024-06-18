Переделайте существующие (или сделайте новые) пять юнит-тестов по предлагаемой методике.

Тесты зависимы от спецификаций.
Можно помещать продукты как во вкусвиле.
Добавляешь продукты в корзину, применяешь купон, и только в этот момент(до оплаты) начинается проверка на наличие товаров на складе(500 рублей скидка на заказ от 3000р, может же быть такое что 1 продукт из за банального отсутсвия на складе), далее 
Или же происходит оплата товара в озоне, и только потом начинается процесс оформления(товар в Китае, логистика итд итп)
# 1. Добавление товара в корзину, когда товар есть в наличии на складе:
```csharp
[Fact]
public void AddItemToCartWhenItemIsInStockIncreasesCartTotal()
{
    var inventory = new Inventory();
    var item = new Item { Id = 1, Name = "Item 1", Price = 100, QuantityInStock = 10 };
    inventory.AddItem(item);
    var cart = new Cart(inventory);
    cart.AddItem(item.Id);
    Assert.Single(cart.Items);
    Assert.Equal(item.Id, cart.Items[0].ItemId);
    Assert.Equal(item.Price, cart.Items[0].Price);
    Assert.Equal(1, cart.Items[0].Quantity);
    Assert.Equal(item.Price, cart.TotalPrice);
}
```
# 2. Когда товара нет в наличии на складе:
```csharp
[Fact]
public void AddItemToCartWhenItemIsOutOfStockThrowsException()
{
    var inventory = new Inventory();
    var item = new Item { Id = 1, Name = "Item 1", Price = 100, QuantityInStock = 0 };
    inventory.AddItem(item);
    var cart = new Cart(inventory);
    Assert.Throws<InvalidOperationException>(() => cart.AddItem(item.Id));
}
```
# 3. Добавление нескольких товаров в корзину, находящихся на складе:
```csharp
[Fact]
public void AddMultipleItemsToCartWhenItemsAreInStockIncreasesCartTotal()
{
    var inventory = new Inventory();
    var item1 = new Item { Id = 1, Name = "Item 1", Price = 100, QuantityInStock = 10 };
    var item2 = new Item { Id = 2, Name = "Item 2", Price = 200, QuantityInStock = 5 };
    inventory.AddItem(item1);
    inventory.AddItem(item2);
    var cart = new Cart(inventory);
    cart.AddItem(item1.Id);
    cart.AddItem(item2.Id);
    Assert.Equal(2, cart.Items.Count);
    Assert.Equal(item1.Price + item2.Price, cart.TotalPrice);
}
```
# 4. Удаление товара из корзины, когда товар есть в наличии на складе:
```csharp
[Fact]
public void RemoveItemFromCart_WhenItemIsInStock_DecreasesCartTotal()
{
    var inventory = new Inventory();
    var item = new Item 
    { 
      Id = 1,
      Name = "Item 1",
      Price = 100,
      QuantityInStock = 10
    };
    inventory.AddItem(item);

    var cart = new Cart(inventory);
    cart.AddItem(item.Id);
    cart.RemoveItem(item.Id);
    Assert.Empty(cart.Items);
    Assert.Equal(0, cart.TotalPrice);
}
```
# 5. Применение скидки к корзине, когда все товары есть в наличии на складе:
```csharp
[Fact]
public void ApplyDiscountToCartWhenAllItemsAreInStockDecreasesCartTotal()
{
    var inventory = new Inventory();
    var item1 = new Item 
    { 
     Id = 1,
     Name = "Item 1",
     Price = 100,
     QuantityInStock = 10
    };
    var item2 = new Item 
    { 
      Id = 2,
      Name = "Item 2",
      Price = 200,
      QuantityInStock = 5
        };
    inventory.AddItem(item1);
    inventory.AddItem(item2);
    var cart = new Cart(inventory);
    cart.AddItem(item1.Id);
    cart.AddItem(item2.Id);
    var discount = new Discount 
    { 
    Code = "гуид",
     Amount = 10 
    };
    cart.ApplyDiscount(discount);
    Assert.Equal(290, cart.TotalPrice);
}
```

Особо практики как таковой не было, поскольку за мои 3 года работы тесты никогда не писались, или тестированием занимались аналитики, или сами разрабы(поднимали вм и мониторили сценарии обработки сообщений от драйвера файловой системы)/
надеюсь что на новой работе мне дадут побольше подобных задач.