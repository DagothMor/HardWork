Самый частый пример - Curiously Recurring Template Pattern

```cs

abstract class Entity<T> where T : Entity<T>
{
    public abstract T Clone();
}

class Person : Entity<Person>
{
    public string Name { get; set; }

    public Person(string name)
    {
        Name = name;
    }
    public override Person Clone()
    {
        return new Person(this.Name);
    }

    public override string ToString() => $"Person: {Name}";
}

```

https://blog.stephencleary.com/2022/09/modern-csharp-techniques-1-curiously-recurring-generic-pattern.html

И тут я задумался, если бы не эта статья и не этот хардворк, как бы я описывал родителя, который может и должен работать только со своими детьми
