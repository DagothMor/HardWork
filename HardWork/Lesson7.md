Вполне возможный кейс на работе:
Представим что у меня есть суперкласс Document который должен выполнять ряд действий описанных в IDocument, среди них это печать.
Я создаю новые классы Excel и Word наследуя от Document и переопределяю метод печать. Но вот может произойти так что в документах есть метки Секретная/Конфедициальная информация, и при печати мне нужно подготовить документ, а именно наложить задний слой диагональной фразой, ФИО печатывающего сотрудника, чтобы отследить утечку распечатанных документов итд. В таком случае придется добавить для каждого класса( у нас же помимо ворда с excel еще и pdf и odt...jpeg в конце концов где логика наложения вотермарки будет абсолютно другой). Получатся громоздкие классы, которые не раз будут переписываться в каждом случае.

Структура реализации через неистинное наследование.
```cs
using System;
using System.Collections.Generic;

namespace TestVisitor
{
    class Program
    {
        static void Main(string[] args)
        {
            var structure = new SpoolerList();
            structure.Add(new WordDocument { });
            structure.Add(new ExcelDocument { });
            structure.PrintAll();// печатать с меткой конфедициальности в шапке документа
            structure.PrintAllConfedential();// печатать с меткой конфедициальности в шапке документа
            structure.PrintAllFIOWithSecretPhrase();// печатать с меткой конфедициальности в шапке документа

            Console.Read();
        }
    }

    // Класс предоставляющий буфер очереди для печати.
    class SpoolerList
    {
        List<IDocument> documents = new List<IDocument>();
        /// <summary>
        /// Добавление в очередь спулера печати.
        /// </summary>
        /// <param name="acc"></param>
        public void Add(IDocument acc)
        {
            documents.Add(acc);
        }
        /// <summary>
        /// Удаление документа из спулера печати.
        /// </summary>
        /// <param name="acc"></param>
        public void Remove(IDocument acc)
        {
            documents.Remove(acc);
        }
        // Распечатать все документы с определенной защитой/паттерном.
        public void PrintAll()
        {
            foreach (IDocument acc in documents)
                acc.Print();
        }
        public void PrintAllConfedential()
        {
            foreach (IDocument acc in documents)
                acc.Print();
        }
        public void PrintAllFIOWithSecretPhrase()
        {
            foreach (IDocument acc in documents)
                acc.Print();
        }
    }

    interface IDocument
    {
        void Open();
        void Close();
        void Save();
        void ChangeMetadata();
        bool IsPrintable();
        void Print();
        void PrintConfedential();
        void PrintFIOWithSecretPhrase();
    }

    abstract class Document : IDocument
    {
        public Guid guid { get; set; }
        public string FilePath { get; set; }

        public void ChangeMetadata()
        {
            throw new NotImplementedException();
        }

        public void Close()
        {
            throw new NotImplementedException();
        }

        public bool IsPrintable()
        {
            throw new NotImplementedException();
        }

        public void Open()
        {
            throw new NotImplementedException();
        }

        public virtual void Print()
        {
            throw new NotImplementedException();
        }

        public virtual void PrintConfedential()
        {
            throw new NotImplementedException();
        }

        public virtual void PrintFIOWithSecretPhrase()
        {
            throw new NotImplementedException();
        }

        public void Save()
        {
            throw new NotImplementedException();
        }
    }

    class WordDocument : Document
    {
        public override void Print()
        {
            throw new NotImplementedException();
        }

        public override void PrintConfedential()
        {
            throw new NotImplementedException();
        }

        public override void PrintFIOWithSecretPhrase()
        {
            throw new NotImplementedException();
        }
    }

    class ExcelDocument : Document
    {
        public override void Print()
        {
            throw new NotImplementedException();
        }

        public override void PrintConfedential()
        {
            throw new NotImplementedException();
        }

        public override void PrintFIOWithSecretPhrase()
        {
            throw new NotImplementedException();
        }
    }
}

```

Вся функциональность печати а так же иные варианты печати вынесены в отдельные классы visitor, что дает нам более чистую логику, не надо менять и расширять реализацию жестко в классах, при новых условиях, достаточно создать новый класс visitor с поведением Y и описать логику для документов формата X


```cs
using System;
using System.Collections.Generic;

namespace TestVisitor
{
    class Program
    {
        static void Main(string[] args)
        {
            var structure = new SpoolerList();
            structure.Add(new WordDocument { });
            structure.Add(new ExcelDocument { });
            structure.PrintAll(new ConfedentialPrintVisitor());// печатать с меткой конфедициальности в шапке документа
            structure.PrintAll(new FIOWithSecretPhrasePrintVisitor());// печатать с ФИО и надписью "СЕКРЕТНО" на заднем плане

            Console.Read();
        }
    }

    interface IVisitor
    {
        // Печать Word Документа.
        void VisitWordDoc(WordDocument acc);
        // Печать Excel Документа.
        void VisitExcelDoc(ExcelDocument acc);
    }

    // Печатать с задником Конфедициально.
    class ConfedentialPrintVisitor : IVisitor
    {
        public void VisitWordDoc(WordDocument document)
        {
            // Модифицируем документ добавляя задний слой с нужным паттерном.
            // document = OpenXML.Layers.Add("Confedential",PaintOverTheCenterBackLayer: true);
            // Передаем модифицированный документ в спулер печати
            // Spool.Print(document);
        }

        public void VisitExcelDoc(ExcelDocument acc)
        {
            // Модифицируем документ добавляя задний слой с нужным паттерном.
            // Избегаем заднее закрашивание ячеек таблицы Excel.
            // document = OpenXML.Layers.Add("Confedential",PaintOverTheCenterBackLayer: false);
            // Передаем модифицированный документ в спулер печати
            // Spool.Print(document);
        }
    }

    // Печатать с задником Секретно и с ФИО печатывающего.
    class FIOWithSecretPhrasePrintVisitor : IVisitor
    {
        public void VisitWordDoc(WordDocument acc)
        {
            // var FIO = OSAgent.GetFIOBySID();
            // Модифицируем документ добавляя задний слой с нужным паттерном.
            // Избегаем заднее закрашивание ячеек таблицы Excel.
            // document = OpenXML.Layers.Add("TOP SECRET" +$"PRINTED BY {FIO}",PaintOverTheCenterBackLayer: true);
            // Передаем модифицированный документ в спулер печати
            // Spool.Print(document);
        }

        public void VisitExcelDoc(ExcelDocument acc)
        {
            // var FIO = OSAgent.GetFIOBySID();
            // Модифицируем документ добавляя задний слой с нужным паттерном.
            // Избегаем заднее закрашивание ячеек таблицы Excel.
            // document = OpenXML.Layers.Add("TOP SECRET" +$"PRINTED BY {FIO}",PaintOverTheCenterBackLayer: false);
            // Передаем модифицированный документ в спулер печати
            // Spool.Print(document);
        }
    }

    // Класс предоставляющий буфер очереди для печати.
    class SpoolerList
    {
        List<IDocument> documents = new List<IDocument>();
        /// <summary>
        /// Добавление в очередь спулера печати.
        /// </summary>
        /// <param name="acc"></param>
        public void Add(IDocument acc)
        {
            documents.Add(acc);
        }
        /// <summary>
        /// Удаление документа из спулера печати.
        /// </summary>
        /// <param name="acc"></param>
        public void Remove(IDocument acc)
        {
            documents.Remove(acc);
        }
        // Распечатать все документы с определенной защитой/паттерном.
        public void PrintAll(IVisitor visitor)
        {
            foreach (IDocument acc in documents)
                acc.Print(visitor);
        }
    }

    interface IDocument
    {
        void Open();
        void Close();
        void Save();
        void ChangeMetadata();
        bool IsPrintable();
        void Print(IVisitor visitor);
    }

    class WordDocument : IDocument
    {
        public string FilePath { get; set; }

        public void Print(IVisitor visitor)
        {
            visitor.VisitWordDoc(this);
        }

        public void ChangeMetadata()
        {
            throw new NotImplementedException();
        }

        public void Close()
        {
            throw new NotImplementedException();
        }

        public bool IsPrintable()
        {
            throw new NotImplementedException();
        }

        public void Open()
        {
            throw new NotImplementedException();
        }



        public void Save()
        {
            throw new NotImplementedException();
        }
    }

    class ExcelDocument : IDocument
    {
        public string FilePath { get; set; }

        public void Print(IVisitor visitor)
        {
            visitor.VisitExcelDoc(this);
        }

        public void ChangeMetadata()
        {
            throw new NotImplementedException();
        }

        public void Close()
        {
            throw new NotImplementedException();
        }

        public bool IsPrintable()
        {
            throw new NotImplementedException();
        }

        public void Open()
        {
            throw new NotImplementedException();
        }

        public void Save()
        {
            throw new NotImplementedException();
        }
    }
}

```

С помощью этого паттерна меньше модифицирую класс, больше получаю возможность расширения функционала. Текущий вариант прекрасно накладывается и на событие Open. Ведь поведение открытия файла с помощью Р7 офис
абсолютно другой, и порядок действий перед открытием(сохранение локальной копии при возможном варианте перетирания тела документа от openxml) может быть абсолютно другим в отличии от базового microsoft office.