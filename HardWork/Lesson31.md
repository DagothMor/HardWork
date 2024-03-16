# 1,2
Метод вытаскивания png из PPT, стал более полиморфным тем, что по сути это сырое вытаскивание, метод которого стал тоже более абстрактным, ведь мы по сути ищем начальный, конечный подмассив. 

```cs
// 1
public static bool ExtractPNGFromRawFile(string path)
        {
            string binaryFilePath = path;
            byte[] fileBytes = File.ReadAllBytes(binaryFilePath);
            byte[] pngSignature = new byte[] { 137, 80, 78, 71, 13, 10, 26, 10 };
            int start = FindSubArray(fileBytes, pngSignature);

            if (start == -1) return false;

            byte[] iendPNGSignature = new byte[] { 0, 0, 0, 0, 73, 69, 78, 68 };
            int end = FindSubArray(fileBytes, iendPNGSignature, start);

            if (end == -1) return false;

            int length = end - start + iendPNGSignature.Length;
            byte[] pngBytes = new byte[length];
            Array.Copy(fileBytes, start, pngBytes, 0, length);

            File.WriteAllBytes("extracted_image.png", pngBytes);

            return true;
        }
		// 2
        static int FindSubArray(byte[] source, byte[] pattern, int start = 0)
        {
            int[] failure = new int[pattern.Length];
            int j = 0;

            for (int i = 1; i < pattern.Length; i++)
            {
                while (j > 0 && pattern[j] != pattern[i])
                {
                    j = failure[j - 1];
                }

                if (pattern[j] == pattern[i])
                {
                    j++;
                }

                failure[i] = j;
            }

            j = 0;
            for (int i = start; i < source.Length; i++)
            {
                while (j > 0 && pattern[j] != source[i])
                {
                    j = failure[j - 1];
                }

                if (pattern[j] == source[i])
                {
                    j++;
                }

                if (j == pattern.Length)
                {
                    return i - pattern.Length + 1;
                }
            }

            return -1;
        }
```

# 3,4,5

## Было
```cs
using ConverterToXml.Converters;
using ImageAndTextExtractorFromBinaryOffice.Extensions;
using ImageMagick;
using NPOI.HSSF.UserModel;
using NPOI.SS.UserModel;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;

namespace ImageAndTextExtractorFromBinaryOffice
{
    class Program
    {
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

            var workingPath = Directory.GetCurrentDirectory();

            var newDirectory = Path.Combine(workingPath, EXTRACT_RESULT_PATH);

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

            var converterDocx2Xml = new DocxToXml();

            var innerTextFromDocx = converterDocx2Xml.ConvertByFile(convertedDocxFilePath);

            using (var writer = new StreamWriter(Path.Combine(newDirectoryForResult, EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION), true))
            {
                writer.Write(innerTextFromDocx);
            }

            return true;
        }
        public static bool ExtractFromXLS(string path, string newFilePath)
        {
            Console.WriteLine("Extracting from xls file...");
            string curDir = Path.GetDirectoryName(path);
            ExtractResult result = new ExtractResult();

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
    }
}


```
## Стало
```cs
using ImageAndTextExtractorFromBinaryOffice.Extensions;
using System;
using System.IO;

namespace ImageAndTextExtractorFromBinaryOffice
{
    class Program
    {
        static void Main(string[] args)
        {
            if (args.Length == 0)
            {
                Console.WriteLine("args is null");
                Console.Read();
                return;
            }
            if (!ArgumentManager.FileIsExistFromStartUpArguments(args, out string filePath))
            {
                Console.WriteLine("File path not found.");
                Console.Read();
                return;
            }
            if (!ArgumentManager.GetOpenFileCommandByExtension(Path.GetExtension(filePath), out Extension supportedExtension))
            {
                Console.WriteLine("Extension was not supported");
                Console.Read();
                return;
            }

            // Debug
            Console.Read();

            FileManager fileManager = new FileManager(filePath,supportedExtension);
            fileManager.Open();
            fileManager.ExtractTextAndImages();
            fileManager.Close();

            Console.Read();

        }
    }
}
```

___

# 3 Возможная реализация вытаскивания информации только одна - вызов метода только открытого файла.
``` cs
using ImageAndTextExtractorFromBinaryOffice.Extensions;
using System;
using System.IO;

namespace ImageAndTextExtractorFromBinaryOffice
{
    public interface IFileState
    {
        bool Open(FileManager context);
        void Close(FileManager context);
        bool ExtractTextAndImages(FileManager context);
    }
    public class FileClosed : IFileState
    {
        public bool Open(FileManager context)
        {
            try
            {
                context.FileStream = new FileStream(context.FilePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
                return false;
            }
            Console.WriteLine("File is opened.");
            context.State = new FileOpen(); // Change state to FileOpen
            return true;
        }

        public void Close(FileManager context)
        {
            Console.WriteLine("File is already closed.");
        }

        public bool ExtractTextAndImages(FileManager context)
        {
            Console.WriteLine("Cannot extract text and images. File is not open.");
            return false;
        }
    }

    public class FileOpen : IFileState
    {
        public bool Open(FileManager context)
        {
            Console.WriteLine("File is already open.");
            // Задумался над этим моментом, вот у открытого файла, есть метод открыть, что он должен вернуть? успешность открытия? а если он уже открыт? 
            return true;
        }

        public void Close(FileManager context)
        {
            Console.WriteLine("File is closed.");
            context.State = new FileClosed();
        }

        public bool ExtractTextAndImages(FileManager context)
        {
            return context.DocumentExtractor.ExtractTextAndImages(context);
        }
    }

    public class FileManager
    {
        public IFileState State { get; set; }
        public Stream FileStream { get; set; }
        public string FilePath { get; set; }
        public Extension FileExtension { get; set; }
        public DocumentExtractor DocumentExtractor { get; set; }
        
        public FileManager(string filePath, Extension extension)
        {
            FileExtension = extension;
            State = new FileClosed();
            FilePath = filePath;
        }

        public void Open()
        {
            State.Open(this);
        }

        public void Close()
        {
            State.Close(this);
        }

        public bool ExtractTextAndImages()
        {
            return State.ExtractTextAndImages(this);
        }
    }
}


```

___

# 4 - Реализация метода только через Поток
# 5 - Неполноценный Pattern Matching
```cs
using ConverterToXml.Converters;
using ImageAndTextExtractorFromBinaryOffice.Extensions;
using ImageMagick;
using NPOI.HSSF.UserModel;
using NPOI.SS.UserModel;
using System;
using System.Collections.Generic;
using System.IO;
using System.Text;

namespace ImageAndTextExtractorFromBinaryOffice
{
    public interface IDocumentExtractor
    {
        public bool ExtractFromDOC(Stream FileStream, string newDirectory);
        public bool ExtractFromXLS(Stream FileStream, string newFilePath);

    }

    public  class DocumentExtractor : IDocumentExtractor
    {
        private const string RESULT_DOCX_FILENAME_AND_EXTENSION = "Result.docx";
        private const string DOCX_MEDIA_PATH = "word/media/";
        private const string EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION = "Result.txt";
        private const string EXTRACT_RESULT_PATH = "Files";

        public bool ExtractTextAndImages(FileManager context)
        {
            var newDirectory = Path.Combine(Directory.GetCurrentDirectory(), EXTRACT_RESULT_PATH);

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

            switch (context.FileExtension)
            {
                case OtherExtension _:
                    return false;
                case DocExtension _:
                    return ExtractFromDOC(context.FileStream,newDirectory);
                case XlsExtension _:
                    return ExtractFromXLS(context.FileStream,newDirectory);
                default:
                    return false;
            }
        }
        public bool ExtractFromDOC(Stream FileStream, string newDirectory)
        {
            Console.WriteLine("Extracting from DOCX file...");

            var ConverterDocToDocx = new ConverterToXml.Converters.DocToDocx();
            MemoryStream convertedDocxMemoryStream;

            string convertedDocxFilePath = Path.Combine(newDirectory, RESULT_DOCX_FILENAME_AND_EXTENSION);

            try
            {
                convertedDocxMemoryStream = ConverterDocToDocx.ConvertFromStreamToDocxMemoryStream(FileStream);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
                return false;
            }

            if (convertedDocxMemoryStream == null)
            {
                return false;
            }

            var listOfExtractedImages = new List<MagickImage>();

            try
            {
                using (var documentArchive = new System.IO.Compression.ZipArchive(convertedDocxMemoryStream))
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
                string imagePath = Path.Combine(newDirectory, extractedMagickImage.FileName);

                extractedMagickImage.Write(imagePath);
            }

            var converterDocx2Xml = new DocxToXml();

            var innerTextFromDocx = converterDocx2Xml.ConvertByFile(convertedDocxFilePath);

            using (var writer = new StreamWriter(Path.Combine(newDirectory, EXTRACTED_TEXT_RESULT_FILENAME_AND_EXTENSION), true))
            {
                writer.Write(innerTextFromDocx);
            }

            return true;
        }
        public bool ExtractFromXLS(Stream FileStream, string newFilePath)
        {
            Console.WriteLine("Extracting from xls file...");
            ExtractResult result = new ExtractResult();

            HSSFWorkbook workbook = new HSSFWorkbook(FileStream);

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

            using (StreamWriter writer = new StreamWriter(Path.Combine(newFilePath, "Files", "Result.txt"), true))
            {
                writer.Write(allText);
            }

            return true;
        }
    }
}

```
Вслед за созданием полиморфного метода который вытаскивает текст и изображения, я семантически разделил свой код на ряд классов: FileManager, ArgumentManager и сам класс DocumentExtractor. Не факт что нам в будущем нужно только вытаскивать информацию из документа, может из бинарного файла нам тоже потребуется вытащить массивы байтов(сжатые картинки PNG из ppt). В таком случае нам достаточно добавить метод в интерфейс IDocumentExtractor и реализовать логику в нашем DocumentExtractor.
Так же метод стал полиморфен в рамках потока, из за разделения обязанности открывать документ(в FileManager), мы принимаем только Memory, отчего код становится гибче и можно работать как через файловую систему, так и через память(легко использовать логику в серверной разработке например). Благодаря материалу "35) Пишем безошибочный код", FileManager стал настоящим FileManager, а не `открыватор,вытащить все,закрыватор`(в примере же это и реализовано, НО), любое расширение работы с файлом мы теперь обязаны прописать во всех состояниях объекта.

# Итог
Пример с 1 и 2 был конечно показательным, но 3 4 5 реализовывались на основе жутких зависимостях от полудохлых библиотек. Если посмотреть как преобразовался код и как мы можем в будущем расширять или, что важнее, переиспользовать логику в других проектах(как например работа с отчетами), то становится понятно преимущество полиморфного кода.

Но есть ли грань? Нужно ли окунаться с головой в эзотерический мир?
Когда нужно писать код как роман? А когда выражениями?
Как вариант - писать литературную оболочку.
```
УзнатьСколькоДетейУкеке(Дерево семейноеДревоКеке)
{
	return Tree.CountNodes(семейноеДревоКеке)
}
```