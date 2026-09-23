# 1

до

```
var ready = orders
    .Where(o => o.IsPaid && o.IsInStock && !o.IsCancelled)
    .ToList();

var blocked = orders
    .Where(o => !o.IsPaid || !o.IsInStock || o.IsCancelled)
    .ToList();
```
Допустим у нас есть ряд микрозаймов которые мы может оформить на сайте, заложив ряд сумм\украшений\движимости\недвижимости

Работая с таким кодом можно будет забыть обновить вторую часть кода после модификации первой.

как решение, выделим условие и в зависимости от прохождения, отправляем покупку\оформление кредита в список валидных\невалидных
 
```cs
public sealed record Order(
    int Id,
    bool IsPaid,
    bool IsInStock,
    bool IsCancelled,
    bool HasDeliveryAddress);

public sealed record ShippingPartition(
    IReadOnlyList<Order> Ready,
    IReadOnlyList<Order> Blocked);

private static bool CanShip(Order order) =>
    order.IsPaid
    && order.IsInStock
    && !order.IsCancelled
    && order.HasDeliveryAddress;

public static ShippingPartition PartitionForShipping(
    IEnumerable<Order> orders)
{
    var ready = new List<Order>();
    var blocked = new List<Order>();

    foreach (var order in orders)
    {
        if (CanShip(order))
        {
	        ready.Add(order);
            continue;
        }
        blocked.Add(order);
    }

    return new ShippingPartition(
        Ready: ready.AsReadOnly(),
        Blocked: blocked.AsReadOnly());
}
```
