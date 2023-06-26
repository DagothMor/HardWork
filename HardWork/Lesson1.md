# Ответ
using System.Collections.Generic;
using System.Threading;
using System;

Избавился от else, инвертировал некоторые if-ы, было очень тяжело ибо нужно было тестировать после модификаций, однако с каждым четким ifом становилось все проще и проще, сужающаяся воронка возможных вариантов виделась все отчетливее.
Избавился там где возможно от проверки на Null, однако есть два куска кода, которые я не модифицировал, поскольку классы, использовались в совсем другой логике.
Process - до

```cs

        /// <summary>

        /// Обработка уведомления от драйвера.

        /// </summary>

        /// <param name="notifyMsg">Сообщение от драйвера.</param>

        /// <returns>Результат обработки.</returns>

        private ProccessingResult Process(NotificationMessage notifyMsg)

{

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. ->");



    ProcessNameChecker processNameChecker;

    try

    {

        processNameChecker = new ProcessNameChecker(notifyMsg.Pid);

    }

    catch (Exception)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            NotAddToCache = true

        };

    }



    if (FileNameChecker.IsFileTemporary(notifyMsg.FileName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName

        };

    }



    // Пропустить обработку сообщения, если процесс, который обращается к файлу из списка необрабатываемых

    bool processNeedToSkip = _processNamesForSkip.Any(p => p == processNameChecker.ProcessName);

    if (processNeedToSkip)

    {

        Logger.Trace($"{nameof(NotificationProcessor)}.{nameof(NeedSkipProcessing)}. Process name found in exception list. Skip. ProcName: {processNameChecker.ProcessName}");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName

        };

    }



    string procName = ProcessName.GetName(notifyMsg.Pid);

    if (procName == null)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            NotAddToCache = true,

            ProcessName = processNameChecker.ProcessName

        };

    }



    if (ApplicationIsNotSupported(notifyMsg, procName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.AppIsNotSupported,

            ProcessName = processNameChecker.ProcessName

        };

    }



    string resultFullFilePath = notifyMsg.FullFilePath;

    bool notSendToWebClient = false;



    ProccessingResult result = ProcessAccessType(ref resultFullFilePath, ref notSendToWebClient, notifyMsg, processNameChecker);

    if (result != null)

    {

        Logger.Info($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. Current process name: {processNameChecker.ProcessName}, pid:{notifyMsg.Pid}.");

        result.ProcessName = processNameChecker.ProcessName;

        return result;

    }



    if (NeedSkipProcessing(resultFullFilePath, processNameChecker.ProcessName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            NotAddToCache = true,

            ProcessName = processNameChecker.ProcessName

        };

    }



    Logger.Info($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. Current process name: {processNameChecker.ProcessName}, pid:{notifyMsg.Pid}.");

    ContextDto context = AgentHelper.GetContext(_contextGetter, notifyMsg);

    if (context == null)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName

        };

    }



    if (!processNameChecker.IsExplorer)

    {

        // Очистить переменные обработки файла проводником.

        Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. Cleare file name history opened by explorer .");

        _explorerOpenFileName = "";

        _explorerOpenFullFilePath = "";

    }



    // Пропустить обработку сообщения, если файл есть в кеше обработанных файлов

    FileAccessCacheElement cacheElement = _processedFilesCache.TryGet(notifyMsg.FullFilePath, EXPLORER_NAME_PROCESS);

    if ((cacheElement != null))

    {

        bool isReadControlAgain = ((notifyMsg.AccessType == eAccessType.ReadControl) && (cacheElement.AccessType == eAccessType.ReadControl));

        if ((notifyMsg.AccessType != eAccessType.ReadControl) || isReadControlAgain)

        {

            eProccessingResult res = ConvertReplyActionToProcessRes(cacheElement.ActionType);

            return new ProccessingResult()

            {

                ActionType = res,

                NotNotifyUser = true,

                NotSendToWebClient = true,

                ProcessName = processNameChecker.ProcessName

            };

        }

    }

    else if (!processNameChecker.IsExplorer)

    {

        if (processNameChecker.IsMsOffice)

        {

            return new ProccessingResult()

            {

                ActionType = eProccessingResult.DefaultAction,

                ProcessName = processNameChecker.ProcessName

            };

        }

    }



    _settingsDto = _clientApiManager.GetSettingsAsync(context.UserSid.Sddl).Result;

    if (_settingsDto == null)

    {

        Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. SettingsDto is null");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

        };

    }



    List<MarkerDto> availableMarks = GetAllowedMarkers(context, notifyMsg);



    // Проверить наличие метки файла в списке доступных пользователю меток          

    MarkCheckResult markCheckResult = IsDocumentsMarkInAvailableList(context, resultFullFilePath, availableMarks, notifyMsg.AccessType, readOnly: false);

    ProccessingResult processResult = ConvertMarkCheckToProcessRes(markCheckResult, context, notSendToWebClient);

    processResult.ProcessName = processNameChecker.ProcessName;

    return processResult;

}

```

___

Process - после

```cs

/// <summary>

/// Обработка уведомления от драйвера.

/// </summary>

/// <param name="notifyMsg">Сообщение от драйвера.</param>

/// <returns>Результат обработки.</returns>

        private ProccessingResult Process(NotificationMessage notifyMsg)

{

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. ->");

    ProcessNameChecker processNameChecker;

    try

    {

        processNameChecker = new ProcessNameChecker(notifyMsg.Pid);

    }

    catch (Exception)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            AddToCache = false,

            SendToWebClient = false,

            NotifyUser = false

        };

    }

    Logger.Info($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. Current process name: {processNameChecker.ProcessName}, pid:{notifyMsg.Pid}.");

    if (ProcessNameNotSupported(notifyMsg.Pid, processNameChecker.ProcessName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.AppIsNotSupported,

            ProcessName = processNameChecker.ProcessName,

            AddToCache = true,

            NotifyUser = true,

            SendToWebClient = false

        };

    }

    // Пропустить обработку сообщения, если процесс, который обращается к файлу из списка необрабатываемых.

    if (_processNamesForSkip.Any(p => p == processNameChecker.ProcessName))

    {

        Logger.Trace($"{nameof(NotificationProcessor)}.{nameof(NeedSkipProcessing)}. Process name found in exception list. Skip. ProcName: {processNameChecker.ProcessName}");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = true,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Пропустить обработку сообщения, если файл шаблонный.

    if (FileNameChecker.IsFileTemporary(notifyMsg.FileName))

    {

        Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(ProcessWithoutAnswere)}. Skip event processing.");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = false,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Пропустить обработку сообщения, если файл из пути для шаблонов.

    if (FileNameChecker.IsFileFromTemplatePath(notifyMsg.FullFilePath))

    {

        Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(ProcessWithoutAnswere)}. Skip event processing.");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = false,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Пропустить обработку сообщения, если файл является исполняемым.

    if (FileNameChecker.IsFileExecutable(notifyMsg.FileName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = false,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Пропустить обработку документов неподдерживаемых форматов.

    if (!FileNameChecker.IsFileFromSupportedDocuments(notifyMsg.FileName))

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = false,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Заблокировать доступ процессу, не предназначенному для обработки документов поддерживаемых форматов.

    if (!processNameChecker.IsForDocumentWork)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.AppIsNotSupported,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = true,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    ContextDto context = AgentHelper.GetContext(_contextGetter, notifyMsg);

    if (context == null)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

            ProcessName = processNameChecker.ProcessName

        };

    }

    _settingsDto = _clientApiManager.GetSettingsAsync(context.UserSid.Sddl).Result;

    if (_settingsDto == null)

    {

        Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. SettingsDto is null");

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.DefaultAction,

        };

    }

    if (processNameChecker.IsExplorer)

    {

        return ProcessAccessType(notifyMsg.FullFilePath, notifyMsg, processNameChecker, context);

    }

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. Clear file full path history opened by explorer: {_explorerOpenFullFilePath}.");

    _explorerOpenFileName = "";

    _explorerOpenFullFilePath = "";

    if (processNameChecker.IsProxy)

    {

        return MarkAuthorization(notifyMsg, processNameChecker, context);

    }

    // Заблокировать доступ процессу, не порожденному прокси.

    if (!processNameChecker.ParentIsProxy)

    {

        return new ProccessingResult()

        {

            ActionType = eProccessingResult.AppIsNotSupported,

            ProcessName = processNameChecker.ProcessName,

            AddToCache = true,

            NotifyUser = true,

            SendToWebClient = false

        };

    }

    // Если пакет офис пытается взаимодействовать с документом впервые.

    if (!_workingProxyCache.workingProxyCache.ContainsKey(processNameChecker.ParentPID))

    {

        var processResult = MarkAuthorization(notifyMsg, processNameChecker, context);

        if (processResult.ActionType == eProccessingResult.Allow)

        {

            Logger.Trace($"##### workingProxyCache.add({processNameChecker.ParentPID}) ,resultFullFilePath1( {notifyMsg.FullFilePath})");

            _workingProxyCache.workingProxyCache.Add(processNameChecker.ParentPID, notifyMsg.FullFilePath);

            AddSendEventMessage(new ClientEventMessage(notifyMsg.FullFilePath, eEventType.OpenDocEvent));

        }

        return processResult;

    }

    // Если пакет офис пытается взаимодействовать с другим документом.

    if (_workingProxyCache.workingProxyCache[processNameChecker.ParentPID] != notifyMsg.FullFilePath)

    {

        Logger.Trace($"##### workingProxyCache.ContainsKey({processNameChecker.ParentPID}) && {_workingProxyCache.workingProxyCache[processNameChecker.ParentPID]} != {notifyMsg.FullFilePath}");

        return new ProccessingResult()

        {

            // TODO: СДЕЛАТЬ НОВЫЙ eProccessingResult.AppInstanceIsBusy

            // А именно: на 1 процесс только 1 экземпляр документа, откройте через проски.

            ActionType = eProccessingResult.AppIsNotSupported,

            ProcessName = processNameChecker.ProcessName,

            NotifyUser = true,

            SendToWebClient = false,

            AddToCache = true

        };

    }

    // Ты что-то упустил?

    Logger.Error($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(Process)}. ##### ???");

    return new ProccessingResult()

    {

        // TODO: СДЕЛАТЬ НОВЫЙ eProccessingResult.AppInstanceIsBusy

        // А именно: на 1 процесс только 1 экземпляр документа, откройте через прокси.

        ActionType = eProccessingResult.AppIsNotSupported,

        ProcessName = processNameChecker.ProcessName,

        NotifyUser = false,

        SendToWebClient = false,

        AddToCache = false

    };

}

```

___
Все еще непонятно касаемо ad hoc полиморфизма, мне нужно было создать абстрактный класс eAccessType, и его наследников(возможные ответы от драйвера windows?)
```cs

                case eAccessType.OpenForRead:

                case eAccessType.OpenForReadWrite:

                    {

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(ProcessAccessType)}. Is OpenForReadWrite operation.");



    // Сохранить имя файла для случая, если потом будет операция копирования или перемещения.

    _explorerOpenFileName = notifyMsg.FileName;

    _explorerOpenFullFilePath = notifyMsg.FullFilePath;

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(ProcessAccessType)}. Save file name for future.");

    return new ProccessingResult()

    {

        ActionType = eProccessingResult.DefaultAction,

        AddToCache = false,

        SendToWebClient = false,

        NotifyUser = false

    };

}

  

                case eAccessType.Rename:

                    {

    Logger.Trace($"[{Thread.CurrentThread.ManagedThreadId}]{nameof(NotificationProcessor)}.{nameof(ProcessAccessType)}. Is Rename operation.");

    return new ProccessingResult()

    {

        ActionType = eProccessingResult.DefaultAction,

        AddToCache = false,

        SendToWebClient = false,

        NotifyUser = false

    };

}

```