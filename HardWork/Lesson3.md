В настройках программы мы можем указывать для какого расширения какая программа должна открываться по умолчанию, так же существует ряд возможных программ через которые пользователь может открывать документ.
Был создан класс который представляет из себя работу со внутренним словарем(где ключ - расширение, а значение - список доступных программ.)
По TDD Обозначим пустой класс(ProgramManager) и метод(ReorderRule), опишу желаемую логику в тестах
```cs
[TestClass]
    public class DefaultLaunchProgramRuleTests
    {
        const string DOCX_EXTENSION = "docx";
        const string DOC_EXTENSION = "doc";
        const string XLSX_EXTENSION = "xlsx";
        const string XLS_EXTENSION = "xls";
        const string PPTX_EXTENSION = "pptx";
        const string PPT_EXTENSION = "ppt";
        const string VSD_EXTENSION = "vsd";
        const string VSDX_EXTENSION = "vsdx";
        const string ODT_EXTENSION = "odt";
        const string ODS_EXTENSION = "ods";
        const string ODP_EXTENSION = "odp";
        const string PDF_EXTENSION = "pdf";

        const string WORD_MO = "word(MO)";
        const string WRITER_LO = "writer(LO)";
        const string MOJO_TEXT = "МойОфис Текст";
        const string R7_OFFICE = "Р7-Офис";
        const string EXCEL_MO = "excel(MO)";
        const string CALC_LO = "calc(LO)";
        const string MINE_OFFICE_NF = "МойОфис Nf";
        const string KBWF = "kbwf";
        const string POWERPOINT_MO = "powerpoint(MO)";
        const string IMPRESS_LO = "impress(LO)";
        const string MINE_OFFICE_PRESENTATION = "МойОфис Презентация";
        const string ACROBAT_RIDER = "Acrobat Rider";
        const string FOXIT_READER = "FoxitReader";
        const string VISIO = "visio";

       [TestMethod]
        public void ReorderRuleTest()
        {
            var defaultLaunchProgramRule = new ProgramManager();
            Assert.IsTrue(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, WORD_MO));
            Assert.AreEqual(WORD_MO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsTrue(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, POWERPOINT_MO));
            Assert.AreEqual(POWERPOINT_MO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsTrue(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, VISIO));
            Assert.AreEqual(VISIO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsTrue(defaultLaunchProgramRule.ReorderRule(PDF_EXTENSION, FOXIT_READER));
            Assert.AreEqual(FOXIT_READER, defaultLaunchProgramRule.DefalutProgram[PDF_EXTENSION].First.Value);
        }

        [TestMethod]
        public void RemoveRuleTest()
        {
            var defaultLaunchProgramRule = new ProgramManager();
            Assert.AreEqual(ACROBAT_RIDER, defaultLaunchProgramRule.GetDefaultProgram(PDF_EXTENSION));
            Assert.AreNotEqual(FOXIT_READER, defaultLaunchProgramRule.DefalutProgram[PDF_EXTENSION].First.Value);
            defaultLaunchProgramRule.RemoveProgram(ACROBAT_RIDER);
            Assert.AreEqual(FOXIT_READER, defaultLaunchProgramRule.DefalutProgram[PDF_EXTENSION].First.Value);
        }
    }
```
Теперь реализую сам класс.

```cs
public class ProgramManager
    {
        static HashSet<string> fileExtensions = new HashSet<string>() {
            "docx","doc","xlsx",
            "xls","pptx","ppt",
            "vsd","vsdx","odt",
            "ods","odp","pdf"};
        static HashSet<string> supportedPrograms = new HashSet<string>() { 
            "word(MO)", "writer(LO)", "МойОфис Текст",
            "Р7-Офис", "excel(MO)", "calc(LO)", "МойОфис Nf",
            "kbwf", "powerpoint(MO)", "impress(LO)",
            "МойОфис Презентация", "Acrobat Rider",
            "FoxitReader", "visio" };
        public Dictionary<string, LinkedList<string>> DefalutProgram;

        public ProgramManager(Dictionary<string, LinkedList<string>> defalutProgram)
        {
            DefalutProgram = defalutProgram;
        }
        public ProgramManager()
        {
            DefalutProgram = new Dictionary<string, LinkedList<string>>(){
            {"docx", new LinkedList<string>( new List<string> { "word(MO)", "writer(LO)", "МойОфис Текст", "Р7-Офис" }) },
            {"odt", new LinkedList<string>( new List<string> { "word(MO)", "writer(LO)", "МойОфис Текст", "Р7-Офис" }) },
            {"doc", new LinkedList<string>( new List<string> { "word(MO)", "writer(LO)", "МойОфис Текст", "Р7-Офис" }) },

            {"xlsx", new LinkedList<string>( new List<string> { "excel(MO)", "calc(LO)", "МойОфис Nf", "kbwf", "Р7-Офис" }) },
            {"ods", new LinkedList<string>( new List<string> { "excel(MO)", "calc(LO)", "МойОфис Nf", "kbwf", "Р7-Офис" }) },
            {"xls", new LinkedList<string>( new List<string> { "excel(MO)", "calc(LO)", "МойОфис Nf", "kbwf", "Р7-Офис" }) },

            {"pptx", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "МойОфис Презентация", "Р7-Офис" }) },
            {"odp", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "МойОфис Презентация", "Р7-Офис" }) },
            {"ppt", new LinkedList<string>( new List<string> { "powerpoint(MO)", "impress(LO)", "МойОфис Презентация", "Р7-Офис" }) },

            {"pdf", new LinkedList<string>( new List<string> {"Acrobat Rider", "FoxitReader" }) },

            {"vsd", new LinkedList<string>( new List<string> {"visio" }) },
            {"vsdx", new LinkedList<string>( new List<string> {"visio" }) }
        };
        }
        /// <summary>
        /// Переопределяет программу по умолчанию
        /// </summary>
        /// <param name="extension">Расширение документа</param>
        /// <param name="program">Программа которая должна стать по умолчанию.</param>
        /// <returns>Успех</returns>
        public bool ReorderRule(string extension,string program)
        {
            if (string.IsNullOrEmpty(extension)|| string.IsNullOrEmpty(program))
            {
                return false;
            }
            if (!fileExtensions.Contains(extension) || !supportedPrograms.Contains(program))
            {
                return false;
            }
            var currentList = this.DefalutProgram[extension];

            if (currentList.Contains(program)) 
            {
                currentList.Remove(program);
            }
            currentList.AddFirst(program);
            return true;
        }

        /// <summary>
        /// Удаляет программу из доступа для всех расширений.
        /// </summary>
        /// <param name="program"></param>
        /// <returns></returns>
        public bool RemoveProgram(string program)
        {
            if (string.IsNullOrEmpty(program))
            {
                return false;
            }
            if (!supportedPrograms.Contains(program))
            {
                return false;
            }
            var currentList = this.DefalutProgram;

            foreach (var extension in this.DefalutProgram)
            {
                extension.Value.Remove(program);
            }
            return true;
        }
        /// <summary>
        /// Добавляет программу для работы с определенным расширением.
        /// </summary>
        /// <param name="extension">Расширение документа</param>
        /// <param name="program">Программа которая должна стать по умолчанию.</param>
        /// <returns>Успех</returns>
        public bool AddRule(string extension, string program)
        {
            if (string.IsNullOrEmpty(extension) || string.IsNullOrEmpty(program))
            {
                return false;
            }
            if (!fileExtensions.Contains(extension) || !supportedPrograms.Contains(program))
            {
                return false;
            }
            var currentList = this.DefalutProgram[extension];

            if (currentList.Contains(program))
            {
                currentList.Remove(program);
            }
            currentList.AddLast(program);
            return true;
        }
        public string GetDefaultProgram(string extension) 
        {
            if (string.IsNullOrEmpty(extension) )
            {
                return String.Empty;
            }
            if (!fileExtensions.Contains(extension) )
            {
                return String.Empty;
            }
            return this.DefalutProgram[extension].First.Value;
        }
    }
```
Логическая архитектура:
У нас есть ряд расширений
```
"docx", "doc", "xlsx" , "xls", "pptx" , "ppt", "vsd", "vsdx", "odt", "ods", "odp", "pdf","vsd", "vsdx"
```
и список поддерживаемых программ.
```
 "word(MO)", "writer(LO)", "МойОфис Текст",
            "Р7-Офис", "excel(MO)", "calc(LO)", "МойОфис Nf",
            "kbwf", "powerpoint(MO)", "impress(LO)",
            "МойОфис Презентация", "Acrobat Rider",
            "FoxitReader", "visio"
```
Но мы должны пресекать тот факт что для docx программа по умолчанию может быть PowerPoint, хоть она и включена в список поддерживаемых программ.
Поэтому напишем дефолтное требование для каждого расширения.
```
docx, odt, doc - word(MO), writer(LO), МойОфис Текст, Р7-Офис
xlsx, ods, xls - excel(MO), calc(LO), МойОфис Nf,kbwf, Р7-Офис
pptx, odp, ppt - powerpoint(MO), impress(LO), МойОфис Презентация, Р7-Офис
pdf - Acrobat Rider, FoxitReader
"vsd","vsdx" - visio
```

Таким образом наш класс, как и тест изменятся, придется добавить новую константу в виде словаря, который будет являться дополнительным фильтром во время изменения списка приложений.
Добавим метод, и используем его в функциях Reorder и Add
```
public bool ProgramIsSupportedByDefaultRule(string extension, string program)
        {
            if (string.IsNullOrEmpty(extension) || string.IsNullOrEmpty(program))
            {
                return false;
            }
            if (!_fileExtensions.Contains(extension) || !_supportedPrograms.Contains(program))
            {
                return false;
            }
            return _defaultRule[extension].Contains(program);
        }
```
```
if (!ProgramIsSupportedByDefaultRule(extension, program)) 
            {
                return false;
            }
```
Логика тестов немного изменилась, по нашей логической архитектуре должно быть понятно, что добавление powerpoint в список поддерживаемых программ для docx невозможно, потому
```
[TestMethod]
        public void ReorderRuleTest()
        {
            var defaultLaunchProgramRule = new ProgramManager();
            Assert.IsTrue(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, WORD_MO));
            Assert.AreEqual(WORD_MO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsFalse(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, POWERPOINT_MO));
            Assert.AreNotEqual(POWERPOINT_MO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsFalse(defaultLaunchProgramRule.ReorderRule(DOCX_EXTENSION, VISIO));
            Assert.AreNotEqual(VISIO, defaultLaunchProgramRule.DefalutProgram[DOCX_EXTENSION].First.Value);

            Assert.IsFalse(defaultLaunchProgramRule.ReorderRule(PDF_EXTENSION, EXCEL_MO));
            Assert.AreNotEqual(EXCEL_MO, defaultLaunchProgramRule.DefalutProgram[PDF_EXTENSION].First.Value);
        }
```
Таким образом при разработке очень важно иметь Дизайн, должно получаться так что первым делом мы пишем дизайн, после тесты, после реализацию. Только после реализации мы должны пройти более тщательное тестирование, и при отлове логических ошибок - расширить/переписать дизайн, в следующую очередь подправить тесты и переписать реализацию, итд. Поскольку всегда будут внешние факторы которые мы не смогли учесть
Например параллельный вызов от Word и Explorer к одному и тому же файлу, от чего может возникнуть гонка.