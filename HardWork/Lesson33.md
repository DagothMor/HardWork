Допустим у нас есть класс Quest, и спустя время мы решили добавить в нашу игру зачарованное оружие(амулет итд...), которое использует заряды, они же в свою очередь восстанавливаются только душами умерших врагов(можно добавить еще что при попытке захватить душу герой временно переносится в иное измерение и сражается с сущностями-конкуррентами) после выполнения какого либо квеста. Проблема в том что ни счетчик душ, ни булеву Согласен_На_Пополнение_Душ в класс квест не реализуешь. Здесь напрашивается паттерн реестр. В качестве централизованного реестра будем использовать SoulCollector.

```cs
namespace FlyWeight
{
    using System;
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Герой, готовы ли вы начать квест? (да/нет)");
            var response = Console.ReadLine();
            if (response?.ToLower() != "да")
            {
                Console.WriteLine("Квест отменен.");
                return;
            }
            Console.WriteLine("Хотите ли вы собирать души для меча во время квеста? (да/нет)");
            response = Console.ReadLine();
            bool collectSouls = response?.ToLower() == "да";
            Sword sword = new Sword();
            Hero hero = new Hero(sword);
            Quest quest = new Quest("Поход против гоблинов", "Уничтожьте гоблинов в лесу");
            if(collectSouls) SoulCollector.Instance.RegisterQuest(quest);
            // Представление убийства гоблинов и сбора душ
            for (int i = 0; i < 3; i++)
            {
                Console.WriteLine($"Гоблин {i + 1} убит.");
                SoulCollector.Instance.CollectSoul(quest);
            }
            
            // Завершение квеста и пополнение меча
            hero.CompleteQuest(quest);
            
            Console.WriteLine($"Квест завершен. Зарядов в мече: {sword.Charges}");
        }
    }

    public class Hero
    {
        public Sword Sword { get; }

        public Hero(Sword sword)
        {
            Sword = sword;
        }

        public void CompleteQuest(Quest quest)
        {
            var soulsCollected = SoulCollector.Instance.GetSoulsCollected(quest);
            Sword.Recharge(soulsCollected);
        }
    }

    public class Sword
    {
        public int Charges { get; private set; } = 0;

        public void Recharge(int souls)
        {
            Charges += souls;
            Console.WriteLine($"{souls} душ(и) перенесено в меч.");
        }
    }

    public class Quest
    {
        public string Title { get; }
        public string Description { get; }
        // Уникальный идентификатор квеста для использования в качестве ключа
        public Guid Id { get; }

        public Quest(string title, string description)
        {
            Title = title;
            Description = description;
            Id = Guid.NewGuid(); // Генерируем уникальный ID для каждого квеста
        }
    }

    // Класс-одиночка для управления сбором душ
    public class SoulCollector
    {
        private static readonly Lazy<SoulCollector> _instance = new Lazy<SoulCollector>(() => new SoulCollector());
        public static SoulCollector Instance => _instance.Value;
        private readonly Dictionary<Guid, int> _soulsCollected = new Dictionary<Guid, int>();
        private SoulCollector() { }
        public void RegisterQuest(Quest quest)
        {
            if (!_soulsCollected.ContainsKey(quest.Id))
            {
                _soulsCollected.Add(quest.Id, 0);
            }
        }

        public void CollectSoul(Quest quest)
        {
            if (_soulsCollected.ContainsKey(quest.Id))
            {
                _soulsCollected[quest.Id]++;
            }
        }

        // Метод для получения количества собранных душ после завершения квеста
        public int GetSoulsCollected(Quest quest)
        {
            if (_soulsCollected.TryGetValue(quest.Id, out var count))
            {
                return count;
            }
            return 0;
        }
    }
}
```
Ой решили добавить в честь китайского нового года босса дракона для ЕЖЕ квестов, ой в честь пасхи нужно добавить возможность согласиться на поиск пасхальных яиц... Проблему конечно мы решили, но этот костыль лишь сигнал о том что придут следующие, а эти сигналы являются явным сигналом о слабости абстракций, которые были спроектированы. Интересен еще момент что после нашей реализации, как это описывать в спецификации, и как ее расширять дальше если потребуется, сложность вырастает в разы.