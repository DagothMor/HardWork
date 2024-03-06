# 1 
Рассмотрим детальнее утилиту для импорта текста и изображений из старых форматов.

```cs
		private const string RESULT_DOCX_FILENAME_AND_EXTENSION = "Result.docx";
        private const string DOCX_MEDIA_PATH = "word/media/";
        private const string EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION = "Result.txt";
        private const string EXTRACT_RESULT_PATH = "Files";

        static void Main(string[] args)
        {
            if (args.Length == 0)
            {
                Console.WriteLine("args is null");
                Console.Read();
                return;
            }
            if (!FileManager.FileIsExistFromStartUpArguments(args, out string filePath))
            {
                Console.WriteLine("File path not found.");
                Console.Read();
                return;
            }
            if (!FileManager.GetOpenFileCommandByExtension(Path.GetExtension(filePath), out Extension supportedExtension))
            {
                Console.WriteLine("Extension was not supported");
                Console.Read();
                return;
            }

            // Debug
            Console.Read();
            
			// 1
            var workingPath = Directory.GetCurrentDirectory();
			// 1
            var newDirectory = Path.Combine(workingPath, EXTRACT_RESULT_PATH);

			// 2
            if (!Directory.Exists(newDirectory))
            {
                DirectoryInfo directoryInfo = Directory.CreateDirectory(newDirectory);
                directoryInfo.Create();

                // https://stackoverflow.com/questions/8821410/why-is-access-to-the-path-denied
                File.SetAttributes(newDirectory, FileAttributes.Normal);
            }
            else
            {
                DirectoryInfo di = Directory.CreateDirectory(newDirectory);
                File.SetAttributes(newDirectory, FileAttributes.Normal);
                di.Delete(true);
                di.Create();
                File.SetAttributes(newDirectory, FileAttributes.Normal);
            }

            switch (supportedExtension)
            {
                case OtherExtension _:
                    break;
                case DocExtension _:
                    ExtractFromDOCX(filePath, newDirectory);
                    break;
                case XlsExtension _:
                    ExtractFromXLS(filePath, newDirectory);
                    break;
                default:
                    break;
            }
        }
```

# 1
Семантически этот код говорит нам что будущие результаты(изображения,текст...) будет храниться в специальной папке, расположенной там где и сам исполняемый файл.
# 2
Этот блок кода отвечает за сброс результатов предыдущей работы программы.

___

```cs
public static bool ExtractFromDOCX(string binaryDocFilePath, string newDirectoryForResult)
        {
            Console.WriteLine("Extracting from DOCX file...");

            var ConverterDocToDocx = new ConverterToXml.Converters.DocToDocx();

            string convertedDocxFilePath = Path.Combine(newDirectoryForResult, RESULT_DOCX_FILENAME_AND_EXTENSION);

            ConverterDocToDocx.ConvertFromFileToDocxFile(binaryDocFilePath, convertedDocxFilePath);

            if (!File.Exists(convertedDocxFilePath))
            {
                return false;
            }

            var listOfExtractedImages = new List<MagickImage>();

			// 3
            try
            {
                using (var documentArchive = new System.IO.Compression.ZipArchive(new FileStream(convertedDocxFilePath, FileMode.Open)))
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
                            catch (Exception exception)
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

			// 4
            var converterDocx2Xml = new DocxToXml();

            var innerTextFromDocx = converterDocx2Xml.ConvertByFile(convertedDocxFilePath);

            using (var writer = new StreamWriter(Path.Combine(newDirectoryForResult, EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION), true))
            {
                writer.Write(innerTextFromDocx);
            }

            return true;
        }
```
# 3
Попытка распаковать документ, проход по файлам из `/media`, попытка обработать изображение(да, на redos/ubuntu да и в принципе в Linux [формат emf не обрабатывается](https://imagemagick.org/script/formats.php)... все это городит конструкцию одной из важных целей метода - вытаскивание изображений.
# 4 
Казалось бы, вроде перевели doc - docx, зачем еще?
Вытащить текст из xml с помощью b2xtranslator намного проще и качественнее...

# 5
```cs
public static bool ExtractFromXLS(string path, string newFilePath)
        {
            Console.WriteLine("Extracting from xls file...");
            string curDir = Path.GetDirectoryName(path);
            ExtractResult result = new ExtractResult();
			// 5
            using FileStream file = new FileStream(path, FileMode.Open, FileAccess.Read);
            HSSFWorkbook workbook = new HSSFWorkbook(file);

            for (int i = 0; i < workbook.NumberOfSheets; i++)
            {
                ISheet sheet = workbook.GetSheetAt(i);

                HSSFPatriarch patriarch = (HSSFPatriarch)sheet.DrawingPatriarch;

                if (patriarch.CountOfAllChildren == 0)
                {
                    continue;
                }

                for (int j = 0; j < patriarch.Children.Count; j++)
                {
                    HSSFPicture picture = null;
                    // 5
                    try
                    {
                        picture = (HSSFPicture)patriarch.Children[j];
                    }
                    catch (Exception)
                    {
                        // дети вытаскиваются не только картинками но и формами.
                        continue;
                    }
                    try
                    {
                        var imageFromBytes = new MagickImage(picture.PictureData.Data);
                        result.ImageItems.Add(
                                       new ImageItem()
                                       {
                                           Name = picture.FileName,
                                           Image = imageFromBytes
                                       });
                    }
                    catch (Exception exception)
                    {
                        result.Errors.Add(
                           new ExtractError()
                           {
                               File = picture.FileName,
                               Error = exception.Message
                           });
                    }
                }
            }

            for (int count = 0; count < result.ImageItems.Count; count++)
            {
                ImageItem imageItem = result.ImageItems[count];
                try
                {
                    string imagePath = Path.Combine(newFilePath, String.IsNullOrEmpty(imageItem.Name)
                        ? count.ToString() + "." + imageItem.Image.Format
                        : imageItem.Name + "." + imageItem.Image.Format);

                    imageItem.Image.Write(imagePath);
                }
                catch (Exception ex)
                {
                    Console.WriteLine(ex);
                    continue;
                }
            }

            var allText = new StringBuilder();

            using FileStream textFile = new FileStream($"Extracted text.txt", FileMode.Create, FileAccess.Write);

            for (int sheetIndex = 0; sheetIndex < workbook.NumberOfSheets; sheetIndex++)
            {
                ISheet sheet = workbook.GetSheetAt(sheetIndex);

                for (int rowIndex = 0; rowIndex <= sheet.LastRowNum; rowIndex++)
                {
                    IRow row = sheet.GetRow(rowIndex);
                    if (row == null) continue;

                    for (int cellIndex = 0; cellIndex < row.LastCellNum; cellIndex++)
                    {
                        ICell cell = row.GetCell(cellIndex);
                        if (cell == null) continue;

                        string cellValue = cell.ToString() + " "; // This gets the text value of the cell
                                                                  // Process the cell value as needed
                                                                  //allText.Append(cellValue + " ");
                        allText.Append(cellValue);
                    }
                }
            }
            textFile.Write(new UTF8Encoding(true).GetBytes(allText.ToString()));

            using (StreamWriter writer = new StreamWriter(Path.Combine(curDir, "Files", "Result.txt"), true))
            {
                writer.Write(allText);
            }

            return true;
        }
```

Вытаскивание текста и изображений сами по себе автономны, однако сам метод возвращает результат в виде класса
```cs
/// <summary>
    /// The result of the extraction
    /// </summary>
    public class ExtractResult
    {
        /// <summary>
        /// File name which was investigated
        /// </summary>
        public string File { get; set; }
        /// <summary>
        /// Image items found
        /// </summary>
        public System.Collections.Generic.List<ImageItem> ImageItems { get; private set; }
        /// <summary>
        /// Errors encountered
        /// </summary>
        public System.Collections.Generic.List<ExtractError> Errors { get; private set; }

        /// <summary>
        /// Default constructor
        /// </summary>
        public ExtractResult()
        {
            this.ImageItems = new System.Collections.Generic.List<ImageItem>();
            this.Errors = new System.Collections.Generic.List<ExtractError>();
        }
    }
```

именно семантика глобального метода, тип результата и оптимизация сложности(не надо будет дважды конвертировать файл в HSSFWorkbook) связывают эти два подметода.

# Итог
Да, есть короткие автономные функции, да есть довольно громоздкие(которые можно выделить в локальный метод, вопрос лишь в связанных переменных), однако из этих маленьких строительных блоков строится блок побольше, цель которого можно понять только(не смотря на имя метода и его комментарий) из композиций меньших.

Нот всего 7, замедли [It's Your Move Diana Ross](https://www.youtube.com/watch?v=vtlgWW8L8f4) и из жанра Synth-pop зародится [vaporwave](https://www.youtube.com/watch?v=aQkPcPqTq4M)
Букв 33.