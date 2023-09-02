
1.1. 

Не смог найти каких либо намеков на подобное, только если гипотетически генерировать множество обьектов и их взаимодействие, но фиксится я как понимаю чисто MOCK классом.
___
1.2. 
Проблема в том что принимая сигнал драйвера мы вызываем метод его обработки, в котором вызывается проверка исполняемого файла(WINWORD.EXE с которым мы работаем или же searchprotocol.exe который мы пропускаем), далее проверка файла(Шаблонный ли что означает с спец символами -$), если процесс от проводника, то погружаемся дальше, какой тип вызова, стоит ли дальше идти проверяя метку документа...
Не вижу иного варианта как переписывать с нуля функционально(опыта пока что к сожалению нет + там столько глобальных переменных и возможных хитросплетений пользователя что нужно нормально погрузиться в это, собственно былоб желанию да ума не хватает:) 
___
1.3.
1 Метод реализует отправку сообщения на сервер, однако eventDTO генерировался внутри метода.
1 До
```cs
public static bool TrySendEvent(
            ClientApiManager clientApiManager,
            eEventType eventType,
            string filePath,
            ContextDto context,
            Guid markerGuid,
            Guid metadataGuid,
            string authorName)
        {
	        // 
	    }
```
1 После
```cs
public static bool TrySendEvent(
            ClientApiManager clientApiManager,
            EventDTO eventdto)
        {
	        // 
	    }
```

2 Метод обрабатывает неправильное слово(ранее оно не было найдено в префиксном дереве выражающем словарь правильных слов), запуская 
рекурсивный обход по тому же префиксному дереву, с рядом начальных значений, которые в дальнейшем будут изменяться().

2 До
```cs
public List<string> GetPossibleCorrections(string wrongWord)
        {
            if (wrongWord == null)
                return null;

            List<string> listWordsWithOneEdit = new List<string>();
            List<string> listWordsWithTwoEdit = new List<string>();

            DFS(root, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit,0,10,0,0);
            return listWordsWithOneEdit.Any() ? listWordsWithOneEdit : listWordsWithTwoEdit;
        }

public void DFS(Node<T> node, string wrongWord, ref List<string> listWordsWithOneEdit, ref List<string> listWordsWithTwoEdit, int charNumber, int lastIter, int deletions, int changes)
        {
            //Console.WriteLine(node.Prefix + "|" + charNumber + "|" + deletions + "|"+ changes);
            if (changes > 2 || node.Used[deletions,changes])
            {
                return;
            }
            node.Used[deletions,changes] = true;
            if (charNumber == wrongWord.Length && node.IsWord) 
            {
                if (changes == 1)
                {
                    listWordsWithOneEdit.Add(node.Prefix);
                }
                else if (changes == 2)
                {
                    listWordsWithTwoEdit.Add(node.Prefix);
                }
            }

            // Удаление
            if (lastIter > 0 && charNumber < wrongWord.Length) 
            {
                DFS(node, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit, charNumber+1, 0,deletions+1,changes+1);
            }
            // Вставка
            if (lastIter > 1)
            {
                foreach (var subnode in node.SubNodes)
                {
                    DFS(subnode.Value, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit, charNumber, 1,deletions,changes+1);
                }
            }
            // Дальше проход.
            if (charNumber < wrongWord.Length && node.SubNodes.ContainsKey(wrongWord[charNumber]))
            {
                DFS(node.SubNodes[wrongWord[charNumber]], wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit, charNumber+1, 2,deletions,changes);
            }
        }
```
После 
```cs
public List<string> GetPossibleCorrections(string wrongWord)
        {
            if (wrongWord == null)
                return null;

            List<string> listWordsWithOneEdit = new List<string>();
            List<string> listWordsWithTwoEdit = new List<string>();
            var accumulator = new DFSAccumulator();
            DFS(root, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit,ref accumulator);
            return listWordsWithOneEdit.Any() ? listWordsWithOneEdit : listWordsWithTwoEdit;
        }
        public void DFS(Node<T> node, string wrongWord, ref List<string> listWordsWithOneEdit, ref List<string> listWordsWithTwoEdit,ref DFSAccumulator accumulator)
        {
            //Console.WriteLine(node.Prefix + "|" + charNumber + "|" + deletions + "|"+ changes);
            if (accumulator.Changes > 2 || node.Used[accumulator.Deletions, accumulator.Changes])
            {
                return;
            }
            node.Used[accumulator.Deletions, accumulator.Changes] = true;
            if (accumulator.CharNumber == wrongWord.Length && node.IsWord) 
            {
                if (accumulator.Changes == 1)
                {
                    listWordsWithOneEdit.Add(node.Prefix);
                }
                else if (accumulator.Changes == 2)
                {
                    listWordsWithTwoEdit.Add(node.Prefix);
                }
            }

            // Удаление
            if (accumulator.LastIter > 0 && accumulator.CharNumber < wrongWord.Length) 
            {
                accumulator.CharNumber += 1;
                accumulator.Deletions += 1;
                accumulator.Changes += 1;
                accumulator.LastIter = 0;
                DFS(node, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit,ref accumulator);
            }
            // Вставка
            if (accumulator.LastIter > 1)
            {
                foreach (var subnode in node.SubNodes)
                {
                    accumulator.Changes += 1;
                    accumulator.LastIter = 1;
                    DFS(subnode.Value, wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit,ref accumulator);
                }
            }
            // Дальше проход.
            if (accumulator.CharNumber < wrongWord.Length && node.SubNodes.ContainsKey(wrongWord[accumulator.CharNumber]))
            {
                accumulator.CharNumber += 1;
                accumulator.LastIter = 2;
                DFS(node.SubNodes[wrongWord[accumulator.CharNumber]], wrongWord, ref listWordsWithOneEdit, ref listWordsWithTwoEdit,ref accumulator);
            }
        }
```
Методы до получались громоздкими и очень легко было ошибиться в правильности порядка, отлаживать было бы той еще мукой, когда ты вроде должен был передать штраф за неправильную букву в удалении, а на самом деле опять же будто ее заменяешь.

К сожалению из за нескольких сценариев хвостовую рекурсию прикрутить не смогу, ибо под двумя модификациями(максимальный штраф 2) имеется в виду удаление И/ИЛИ изменение 
___

1.4. 

 Когда велась разработка класса FileManager. У IFileManager был только единственный метод Open(), отвечающий за открытие документа. В этом методе при отсутствии метадаты у открываемого документа, он генерировал новую. Только недавно пришлось реализовывать альтернативу создания документа через ПКМ, там же интерфейс IFileManager был расширен методом GenerateNewMetadata, таким образом уменьшилось сложность чтения кода, ибо непонятно открываем ли мы документ, либо специально проставляем метку.
До
```cs
 using (IFileManager file = new FileManager(TEMPLATE_DOCX_PATH, _currentContext, needSaveToFile: true, needCreatePublicMarker: true))
            {
            try
            {
	            file.Open();
            }
	        catch (Exception ex)
            {
	            //Logger.Error($"ERROR: {ex}");
	            return false;
            }
            finally
            {
	            return true;
            }
            }
```
 После
```cs
            using (IFileManager file = new FileManager(TEMPLATE_DOCX_PATH, _currentContext, needSaveToFile: true, needCreatePublicMarker: true))
            {
                var success = file.CreateMetadataForTemplateDocument();
                if (!success)
                {
                    //Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}. ##### SOMETHING WAS WRONG.");
                    return false;
                }
            }

```
1.5. Чрезмерный результат. Метод возвращает больше данных, чем нужно вызывающему его компоненту.

Тоже не нашел к сожалению ярких примеров. На ум приходит только злоупотребление выходным параметром out. Правильное применение тот же Int32.TryParse(string numbers,out int number). Этот метод не только говорит нам об успешности операции, но и результатом парсинга. Подобное я реализовывал при обработке аргументов при вызове приложения.
```cs
/// <summary>
        /// Если нету аргумента с существующим файлом, то уведомляем.
        /// </summary>
        /// <param name="FilePath">Путь к документу.</param>
        /// <param name="exePath">Путь к исполняемому файлу.</param>
        /// <returns>Путь к исполняемому файлу существует.</returns>
        public static bool ExecutablePathIsExist(string FilePath, out string exePath)
        {
            var executablePath = RegistryHelper.GetExecutablePath(FilePath);
            if (String.IsNullOrEmpty(executablePath))
            {
                //Logger.Error($"Exe in {executablePath} not found.");
                MessageBox.Show("Exe not found.", "Error", MessageBoxButton.OK, MessageBoxImage.Error);
                exePath = String.Empty;
                return true;
            }
            exePath = executablePath;
            return false;
        }
```