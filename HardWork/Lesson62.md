Питон тяжело понимается, особенно передача полей и их взаимодействие внутри, когда прямых вызовов я не вижу, предлагаю вариант на C#, с помощью шарповского yield
```cs
static void Main(string[] args)
{
    PrintFibonacci();
    Console.ReadLine();
}

static void PrintFibonacci()
{
    Console.WriteLine("Fibonacci numbers:");

    foreach (int number in GetFibonacci(1000))
    {
        Console.WriteLine(number);
    }
}

static IEnumerable<int> GetFibonacci(int maxValue)
{
    int previous = 0;
    int current = 1;

    while (current <= maxValue)
    {
        yield return current;

        int newCurrent = previous + current;
        previous = current;
        current = newCurrent;
    }
}
```
