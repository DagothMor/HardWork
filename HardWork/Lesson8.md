Пример через расширение методов.
Методы расширения можно использовать для расширения класса или интерфейса, но не для их переопределения.
Во время компиляции методы расширения всегда имеют более низкий приоритет, чем методы экземпляра, определенные в самом типе, таким образом они вполне безопасны.

```cs
namespace ExtensionMethodSample
{
    using System;

    public interface IDocument
    {
        void Open();
    }

    // Расширенные методы могут быть доступны через экземпляры обьектов которые реализуют IDocument.
    public static class Extension
    {
        public static void Copy(this IDocument doc, int CountOfCopies)
        {
            Console.WriteLine
                ("COPY N TIMES");
        }

        public static void Copy(this IDocument doc, string secretKey)
        {
            Console.WriteLine
                ("COPY WITH SECRET KEY");
        }

        // Этот метод никогда не вызовится поскольку во всех классах этот метод реализован.
        public static void Open(this IDocument doc)
        {
            Console.WriteLine
                ("Its never called.");
        }
    }

    class WordDocument : IDocument
    {
        public void Open() { Console.WriteLine("Opening Word document"); }
    }

    class ExcelDocument : IDocument
    {
        public void Open() { Console.WriteLine("Opening Excel Document"); }
        public void Copy(int countOfCopies) { Console.WriteLine("Copy Excel Document many times"); }
    }

    class PDFDocument : IDocument
    {
        public void Open() { Console.WriteLine("Opening PDF document"); }

        public void Copy(int countOfCopies)
        {
            Console.WriteLine("Copy PDF Document many times");

        }
        public void Copy(object obj)
        {
            Console.WriteLine("Copy PDF Document with obj parametr.");
        }
    }

    class ExtMethodDemo
    {
        static void Main(string[] args)
        {
            WordDocument wordDocument = new WordDocument();
            ExcelDocument excelDocument = new ExcelDocument();
            PDFDocument pdfDocument = new PDFDocument();

            wordDocument.Copy(1);           // Метод через расширение.
            wordDocument.Copy("Pavel P.");     // Метод через расширение.

            wordDocument.Open();            // Реализация в классе

            excelDocument.Copy(1);           // Реализация в классе
            excelDocument.Open();            // Реализация в классе

            excelDocument.Copy("Alexei S.");     // Метод через расширение

            pdfDocument.Copy(1);           // Реализация в классе
            pdfDocument.Copy("Grisha P.");     // Реализация в классе
            pdfDocument.Open();            // Реализация в классе
            Console.Read();
        }
    }
}
```

Таким образом можно расширить функционал через Extension класс, однако мы привязаны к созданным методам, и интерфейсам. Может получится такая ситуация когда придется создавать новый интерфейс и наследовать для тех классов с которыми мы хотим работать.