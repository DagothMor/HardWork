# 2 Библиотеки для вытаскивания текста и изображений для старых форматов.
## 2.1 Проблема платформы
b2x работоспособен только на платформе .NET Core 3.1, его обновление вызывает краш, причем в очень интересном месте
[месте](https://github.com/EvolutionJobs/b2xtranslator/blob/90d05a6589706cf177a245fcb74e9cba4b6264ae/Doc/WordprocessingMLMapping/MainDocumentMapping.cs#L40C21-L40C51)

```
https://habr.com/ru/companies/auriga/articles/528084/

Если вы используете платформу _.Net Core 3_ и выше в своем решении, обратите внимание на целевые среды для подключенных проектов _b2xtranslator_. Так как библиотека была написана довольно давно и не обновляется с 2018 года, по умолчанию она собирается под .Net Core 2.  
Чтобы сменить целевую среду, щелкните правой кнопкой мыши по проекту, выберите пункт «Свойства» и поменяйте целевую рабочую среду. В противном случае вы можете столкнуться с проблемой невозможности конвертации файлов .doc, содержащих в себе таблицы.
```

Оставим как есть, все равно будем собирать библиотеку-солянку.

# 2.2 Отсутствие поддержки ppt -> pptx в b2xTranslator и NPOI
Нет, серьезно.

https://t.me/npoidevs
введите поиск в чате ppt и сами удостоверьтесь, doc и ppt не поддерживаются NPOI, хотя [написано](https://github.com/tonyqus/npoi)

```
With NPOI, you can read/write Office 2003/2007 files very easily.
```

Ни мелкого шрифта, ни звездочки...

в b2xTranslator функционал не доделан. Можно конечно погрузиться во все это вот, однако время ограничено + постоянные сложные ошибки.

# 2.3 Получение текста из doc -> docx через b2x
Да, текст вытаскивается, к сожалению буквально, а именно с тегами. Дополнительную сложность дают BOM символы для парсинга, регулярку нашли, однако еще нужно протестить, а целостная ли структура по стандарту OpenXML.

# Итог
Чем проблема узконаправленнее , тем уже функционал, лицензии и реализации, и тем глубже приходится копаться, вплоть до китайской документации и чатов.
# 3 

## 3.1 ExtractResult

Созданы методы в библиотеке(Объединены текст и картинки, ибо на 10000 документов пришлось бы открывать 20000 MemoryStream)
```cs
public static ExtractResult GetTextAndImagesFromDoc(Stream docStream)
public static ExtractResult GetTextAndImagesFromXls(Stream docStream)
```
Сама библиотека будет внедряться в другую, мной составлен возвращаемый тип.
```cs
    public class ExtractResult
    {
        public string File { get; set; }
        public string Text { get; set; }
        public System.Collections.Generic.List<Exception> textExceptions { get; private set; }
        /// <summary>
        /// Список вырезанных картинок.
        /// </summary>
        public System.Collections.Generic.List<ImageItem> ImageItems { get; private set; }
        /// <summary>
        /// Словарь ошибок для исключений.
        /// </summary>
        public System.Collections.Generic.Dictionary<string,Exception> imageErrors { get; private set; }

        public ExtractResult()
        {
            this.ImageItems = new System.Collections.Generic.List<ImageItem>();
            this.imageErrors = new System.Collections.Generic.Dictionary<string, Exception>();
            this.textExceptions = new List<Exception>();
        }
    }
```

Таким образом для легкой адаптации можно использовать мой класс, или на крайний случай создать множество и передать его в библиотеку выше. Данные вернутся по максимуму, не только для бизнес логики, но и для отлаживания, так без проблем сможем мониторить статистику неудачных вытаскиваний картинок и текста из ячеек.

# 3.2 Добавление отлова исключений в форке от b2xtranslator
```cs
public string Convert(Stream memStream,out List<Exception> exceptions)
        {
            exceptions = new List<Exception>();
            Dictionary<int, string> listEl = new Dictionary<int, string>();

            string xml = string.Empty;
            memStream.Position = 0;
            using (WordprocessingDocument doc = WordprocessingDocument.Open(memStream, false))
            {
                StringBuilder sb = new StringBuilder(1000); // врядли в xml будет меньше 1000 символов
                sb.Append("<?xml version=\"1.0\"?><documents><document>");
                Body docBody = doc.MainDocumentPart.Document.Body; // тело документа (размеченный текст без стилей)
                CreateDictList(listEl, docBody);
                foreach (var element in docBody.ChildElements)
                {
                    string type = element.GetType().ToString();
                    try
                    {
                        switch (type)
                        {
                            case "DocumentFormat.OpenXml.Wordprocessing.Paragraph":

                                if (element.GetFirstChild<ParagraphProperties>() != null && element.GetFirstChild<ParagraphProperties>()
                                    .GetFirstChild<NumberingProperties>() != null) // список / не список
                                {
                                    if (element.GetFirstChild<ParagraphProperties>().GetFirstChild<NumberingProperties>().GetFirstChild<NumberingId>().Val != CurrentListID)
                                    {
                                        CurrentListID = element.GetFirstChild<ParagraphProperties>().GetFirstChild<NumberingProperties>().GetFirstChild<NumberingId>().Val;
                                        sb.Append($"<li id=\"{CurrentListID}\">");
                                        InList = true;
                                        ListParagraph(sb, (Paragraph)element);
                                    }
                                    else // текущий список
                                    {
                                        ListParagraph(sb, (Paragraph)element);
                                    }
                                    if (listEl.ContainsValue(((Paragraph)element).ParagraphId.Value))
                                    {
                                        sb.Append($"</li id=\"{element.GetFirstChild<ParagraphProperties>().GetFirstChild<NumberingProperties>().GetFirstChild<NumberingId>().Val}\">");
                                    }
                                    continue;
                                }
                                else // не список
                                {
                                    SimpleParagraph(sb, (Paragraph)element);
                                    continue;
                                }
                            case "DocumentFormat.OpenXml.Wordprocessing.Table":

                                Table(sb, (Table)element);
                                continue;
                        }
                    }
                    catch (Exception e) // В случае наличия в документе тегов отличных от нужных, они будут проигнорированы
                    {
                        exceptions.Add(e);
                        continue;
                    }
                }
                sb.Append(@"</document></documents>");
                xml = sb.ToString();
            }
            return xml;
        }
```

Теперь string Convert возвращает не только xml от потока doc файла, но и список исключений, возникающих при конвертации xml элементов.
