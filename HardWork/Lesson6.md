___
# 1
### Идея/Спецификация
Администратор должен настраивать для сотрудника в настройках список доступных приложений с помощью которых пользователь может открывать документы.

### Были добавлены тесты с шаблонными ассертами
С помощью предыдущих уроков было легче писать тесты, поскольку Спецификация написана и правильно поставлены вопросы что должно делаться, а что НЕ должно сделаться. 

```cs
[TestMethod]
        public void DeleteExecutableFileFromAccessListByExtension()
        {
            var defaultRule = new ProgramManager();

            // Удаление приложения "" для расширения"".
            Assert.AreEqual(false, false);
            // Удаление выдуманного приложения для выдуманного расширения.
            Assert.AreEqual(false, false);
            // Удаление существующего приложения для существующего расширения.
            Assert.AreEqual(true, true);
            // Удаление несуществующего приложения для существующего расширения.
            Assert.AreEqual(false, false);
        }

        [TestMethod]
        public void AddExecutableFileToAccessListByExtension()
        {
            var defaultRule = new ProgramManager();

            // Добавление приложения "" для расширения"".
            Assert.AreEqual(false, false);
            // Добавление выдуманного приложения для выдуманного расширения.
            Assert.AreEqual(false, false);
            // Добавление существующего приложения для существующего расширения.
            Assert.AreEqual(true, true);
            // Добавление УЖЕ существующего приложения для существующего расширения.
            // Должно возвращать true.
            Assert.AreEqual(true, true);
        }

        [TestMethod]
        public void ExtensionAndExecutableFileAreCorrect()
        {
            var defaultRule = new ProgramManager();

            // Дефолтное правило содержит docx.
            Assert.AreEqual(true, true);
            // Дефолтное правило содержит odt.
            Assert.AreEqual(true, true);
            // Дефолтное правило содержит mp4.
            Assert.AreEqual(false, false);
        }
```

### Шаг за шагом добавлялась логика
Чтобы не вставлять в нескольких местах и легче отлаживать код, я объединил удаление/добавление операцией модификация, иначе создал вспомогательную функцию ChangeApplicationRule которая дополнительно фильтрует на вход расширение и исполняемый файл.
```cs
		/// <summary>
        /// Изменение администратором правила открытия документов.
        /// </summary>
        /// <param name="extension">Ключ правила.</param>
        /// <param name="executableFile">Значение которое нужно удалить\добавить.</param>
        /// <param name="isDelete">Удаление, иначе добавление.</param>
        /// <returns>Возвращает успешность операции.</returns>
        public bool ChangeApplicationRule(string extension, string executableFile, bool isDelete)
        {
            return !ExtensionAndExecutableFileAreCorrect(extension, executableFile)
                ? false
                : isDelete
                    ? DeleteExecutableFileFromAccessListByExtension(extension, executableFile)
                    : AddExecutableFileToAccessListByExtension(extension, executableFile);
        }
		/// <summary>
        /// Расширение и исполняемый файл корректны для текущего обьекта.
        /// </summary>
        /// <param name="extension">Расширение документа.</param>
        /// <param name="executableFile">Имя исполняемого файла.</param>
        /// <returns>Возвращает успешность операции.</returns>
        public bool ExtensionAndExecutableFileAreCorrect(string extension, string executableFile)
        {
            if (string.IsNullOrEmpty(extension) || string.IsNullOrEmpty(executableFile))
            {
                return false;
            }
            if (!_fileExtensions.Contains(extension) || !_supportedPrograms.Contains(executableFile))
            {
                return false;
            }

            return true;
        }

        /// <summary>
        /// Удаление приложения из списка доступных в правиле для расширения файла.
        /// </summary>
        /// <param name="extension">Ключ правила.</param>
        /// <param name="executableFile">Значение которое нужно удалить\добавить.</param>
        /// <param name="isDelete">Будет происходить операция удаления?</param>
        /// <returns>Возвращает успешность операции.</returns>
        public bool DeleteExecutableFileFromAccessListByExtension(string extension, string executable)
        {
            bool success = false;
            
            return success;
        }

        /// <summary>
        /// Добавление приложения в список доступных в правиле для расширения файла.
        /// </summary>
        /// <param name="extension">Ключ правила.</param>
        /// <param name="executableFile">Значение которое нужно удалить\добавить.</param>
        /// <param name="isDelete">Будет происходить операция удаления?</param>
        /// <returns>Возвращает успешность операции.</returns>
        public bool AddExecutableFileToAccessListByExtension(string extension, string executable)
        {
            bool success = false;

            return success;
        }
```
Изменения в коде были действительно маленькими, заметил интересный ход, касаемо использования вспомогательной функции. У меня могло быть и больше функций(авторизованное создание/удаление/вытаскивание возможных программ у клиента для проверки, а почему у него приложение Х, хотя вся политика компании его запретила в рабочих процессах), однако сам факт существования вспомогательной немного изменил подход к оформлению тестов и самой реализации.

# 2 
Второй опыт был не столь удачным.

Создание документа должно выполняться специальным приложением Y, поскольку мы должны пропускать любой перехват событий от файловой системы для Y.GetProcessName() .
Проблема была в том что тестирование для этой задачи не напишешь, отлаживать нужно будет вручную, устанавливая наработки и библиотеки непосредственно в службу и отлаживать пошагово.
Были добавлены скрытое поле имени процесса Y, его Get() свойство для класса ProcessNameChecker, добавлена логика в pipeline мониторинга событий файловой системы... А оказывается что за прописанную в Y работу с файлом(удаление шаблонного файла и копирование из внутренней папки программы в то же место) отвечает explorer.exe(хоть и дочерний от Y.exe, отчего в дальнейшем я отталкивался)... Я знал что мог бы отлаживать меньшими шагами, но я бы не понял тот факт, что все шаги в тесте равны, а есть ровнее... Если бы я знать как задать себе правильные вопросы во время составления логического дизайна, то я бы написал болванку в виде ехе который создает пустой блокнот через streamwriter, и посмотрел бы, а точно мой процесс стучится к файлу? не explorer?

___
Если задача тривиальна, то я коммичу раз в день, комментируя что успел добавить/реализовать
Если задача нетривиальна, то я коммичу раз в пару часов описывая текущую ситуацию, потому что неизвестно, оправдаются ли мои догадки, или же придется откатываться назад и идти другим путем.