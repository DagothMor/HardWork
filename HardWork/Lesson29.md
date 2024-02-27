Написанная утилита должна вытаскивать картинки и текст из бинарных документов.
Контекст: только opensource с лицензией аля apache 2.0, поскольку продукт коммерческий и лицензии дорогие, забугорные. Работать должна на linux системах.
# 1 До
```cs
public static bool ExtractFromDOCX(string binaryDocFilePath,string newDirectoryForResult)
        {
            var ConverterDocToDocx = new ConverterToXml.Converters.DocToDocx();
            string convertedDocxFilePath = Path.Combine(newDirectoryForResult, RESULT_DOCX_FILENAME_AND_EXTENSION);
            ConverterDocToDocx.ConvertFromFileToDocxFile(binaryDocFilePath,convertedDocxFilePath);
            if (!File.Exists(convertedDocxFilePath))
            {
                return false;
            }
            var listOfExtractedImages = new List<MagickImage>();
            try
            {
                using (var documentArchive = new System.IO.Compression.ZipArchive(new FileStream(convertedDocxFilePath , FileMode.Open)))
                {
                    foreach (System.IO.Compression.ZipArchiveEntry entry in documentArchive.Entries)
                    {
                        if (entry.FullName.StartsWith(DOCX_MEDIA_PATH))
                        {
                            try
                            {
                                var imageFromStream = new MagickImage(entry.Open());
                                listOfExtractedImages.Add(imageFromStream);
                            }
                            catch (System.Exception exception)
                            {
                                Console.WriteLine(exception);
                            }
                        }
                    }
                }
            }
            catch (Exception exception)
            {
                Console.WriteLine(exception);
            }
            foreach (var extractedMagickImage in listOfExtractedImages)
            {
                string imagePath = Path.Combine(newDirectoryForResult, extractedMagickImage.FileName);
                extractedMagickImage.Write(imagePath);
            }
            var converterDocx2Xml = new DocxToXml();
            var innerTextFromDocx = converterDocx2Xml.ConvertByFile(convertedDocxFilePath);
            using (var writer = new StreamWriter(Path.Combine(newDirectoryForResult, EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION), true))
            {
                writer.Write(innerTextFromDocx);
            }
            return true;
        }
```
Код раздут, здесь логика и конвертирования, и вытаскивания картинок и текста, так же нейминг метода хромает, будто просто распаковка docx файла.

После уже подразумевает адаптацию под библиотеку, а не под утилиту, вертеться будет на сервере, следовательно сохранять в файл не имеет смысла, работаем только с потоком, в качестве результата будет не булева(успех), а объект класса ExtractResult, который включает в себя текст, ошибка при вытаскивании текста, список (ошибок-имя изображения) при вытаскивании изображений(для сбора статистики, сами исключения мы будем добавлять в логгер непосредственно в методе, вызывающем пример выше из библиотеки) и сам список изображений, таким образом мы можем расширить класс ExtractResult, добавив два поля например метаданные файла, и ошибка при вытаскивании метаданных.

Выделим в локальные методы, которые говорят сами за себя
# После

```cs
public static ExtractResult GetTextAndImagesFromDoc(Stream docStream)
        {
            MemoryStream docxStream = ConvertDocStream(docStream);

            ExtractResult result = new ExtractResult();

            ExtractImagesFromDOCX(docxStream, result);
            ExtractTextFromDOCX(docxStream, result);
            //...
            return result;
        }
```
Не нужно беспокоиться о зависимых переменных(местоположение нового файла) во время 
# 2 До
Допустим у нас есть три метода возможной конвертации из doc в docx 
1 - преобразование из пути исходного файла в путь к новому файлу
2 - получить поток преобразованного файла из пути исходного
3 - получить конвертированный поток из потока исходного файла   

Глупым способом будет реализация(копирования) логики для каждого метода.

# После
используем логику поток-поток как основную, для того чтобы методы поток-файл и файл-файл использовали ее.

```cs
public void ConvertFromFileToDocxFile(string docPath, string docxPath)
        {
            using (FileStream fs = new FileStream(docPath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite))
            {
                MemoryStream docxMemoryStream = ConvertFromStreamToDocxMemoryStream(fs);
                using (FileStream docxFileStream = new FileStream(docxPath, FileMode.OpenOrCreate))
                {
                    docxFileStream.Write(docxMemoryStream.ToArray());
                }
            }
        }
        public MemoryStream ConvertFromFileToDocxMemoryStream(string docPath)
        {
            using (FileStream fs = new FileStream(docPath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite))
            {
                return ConvertFromStreamToDocxMemoryStream(fs);
            }
        }
        public MemoryStream ConvertFromStreamToDocxMemoryStream(Stream stream)
        {
            StructuredStorageReader reader = new StructuredStorageReader(stream);
            WordDocument doc = new WordDocument(reader);
            var docx = WordprocessingDocument.Create("docx", DocumentType.Document);
            Converter.Convert(doc, docx);
            return new MemoryStream(docx.CloseWithoutSavingFile());
        }
```
# 3 До
В проекте была огромная логика не только проверки на существование полей сообщения, но и для особенных случаев по разным видам события.
```cs
        private void SendEventMessageToServer(ClientEvent eventMessage)
        {
	        // Check eventMessage for incorrect values...
	        
            // если документ переименовали, (или перетащили?)то переименуем перед отправкой.
            if (eventMessage.DocEvent.Equals(enumEventType.DocIsRenamed) || eventMessage.DocEvent.Equals(enumEventType.DocIsMoved))
            {
                eventMessage.FileName = eventMessage.CloningFileName;
            }
			if (eventMessage.DocEvent.Equals(enumEventType.DocIsDeleted))
            {
                // Some logic
            }
            if (eventMessage.DocEvent.Equals(enumEventType.DocIsCopied))
            {
                // Some logic
            }
            //...
            // SendMessage...
        }
```

# 3 После
Решением будет ad hoc полиморфизм, создадим подклассы ClientEvent, где имя типа будет характеризовать события над файлом
```cs
private void SendEventMessageToServer(ClientCopiedEvent eventMessage)
        {
			// Check eventMessage for incorrect values...
			// OtherLogic();
			// SendMessage...
		}

private void SendEventMessageToServer(ClientRemovedEvent eventMessage)
        {
			// Check eventMessage for incorrect values...
			// OtherLogic();
			// SendMessage...
		}

private void SendEventMessageToServer(ClientMovedEvent eventMessage)
        {
			// Check eventMessage for incorrect values...
			// OtherLogic();
			// SendMessage...
		}
private void SendEventMessageToServer(ClientCreateEvent eventMessage)
        {
			// Check eventMessage for incorrect values...
			// OtherLogic();
			// SendMessage...
		}
		//...
```

Таким образом нам не нужно будет городить большое количество условий(событий может быть не 10 а более ) для специфичных значений поля перечисления.

# 4 WinApi
Буквально в последнем своем [посте](https://matkarimovalexander.github.io/2024/02/25/Writing-An-Active-Window-Hook.html) я использовал подобный метод.
```cs
[DllImport("user32.dll")]
    public static extern IntPtr SetWinEventHook(uint eventMin, uint eventMax, IntPtr hmodWinEventProc, WinEventDelegate lpfnWinEventProc, uint idProcess, uint idThread, uint dwFlags);
```
Там как раз указаны какие значения для полей я использовал отталкиваясь от предыдущих.

Решением будет разбиение метода SetWinEventHook на более узкие

Сможем убрать случай когда idProcess и idThread равны 0
```cs
public static extern IntPtr SetWinEventHookFromAllProcessesAndThreads(uint eventMin, uint eventMax, IntPtr hmodWinEventProc, WinEventDelegate lpfnWinEventProc, uint dwFlags);
```
Избавились от IntPtr hmodWinEventProc поскольку работаем вне контекста
```cs
[DllImport("user32.dll")]
    public static extern IntPtr SetWinEventHookOutOfContext(uint eventMin, uint eventMax, WinEventDelegate lpfnWinEventProc, uint idProcess, uint idThread, uint dwFlags);
```
Избавились от uint eventMin, uint eventMax, выделив случай для EVENT_SYSTEM_FOREGROUND = 3
```cs
[DllImport("user32.dll")]
    public static extern IntPtr SetWinEventHookActiveWindowIsChanged( WinEventDelegate lpfnWinEventProc, uint idProcess, uint idThread, uint dwFlags);
```
итд...

Итогом будет то что не стоит городить супер-метод, он, подобно опухоли, может разрастаться новой логикой для определенного значения или комбозначений, создавая невидимую сложность для разработчика, повышая сложность понимания самого метода в разы.