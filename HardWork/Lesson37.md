﻿# 1 FileManager
В проекте был fileManager отвечающий за открытие/закрытие файла и CRUD операции над метаданными файла.

Нужно было расширить логику, ибо этот класс использовался и в отдельном приложении, который вызывается автоматом при открытии документов. Может возникнуть такая ситуация, что файл в это время может быть заблокирован другим процессом.
Так же в операцию открытия автоматически была попытка открыть метаданные, если их не было то создавались новые и вшивались в нее. На самом деле это лишь по итогу мешало, поскольку файл при создании от утилиты и так это собирался делать.

По итогу переделал под конечный автомат, разделил логику на атомарные операции и добавил количество попыток открытия через N миллисекунд.

# 2 Раздутие логики создания http запроса
После того как мы поняли что чтото с файлом произошло, в методе создания запроса мы анализировали что именно случилось с файлом, и в зависимости от видов копирования, удаление, переименования, мы меняли поля объекта, который в последствии сериализуется и отправится на сервер. Но и этим дело не кончалось, так же еще очищался/обновлялся кеш открывшихся недавно файлов, поддерживание кода усложнялось.

решение было добавление фильтра перед созданием запросов и промежуточный слой, который выполнял работу непосредственно с кешем открытых документов.

# 3 Мастер документ

Для файла было много методов и условий касаемо вытаскивания картинок и текста, так же в рамках жизни прототипа, многие методы были не реализованы и кидали исключения.
Решением было выделением документа в АТД, наследниками были классы бинарные документы(старого формата) и современные(OpenXML), сам ряд методов по работе с вытаскиванием картинок и текста , а так же CRUD операции с метаданными были выделены в отдельные интерфейсы и комбоинтерфейс, для удобства работы.

# 4 Отмена или продолжить
Есть очень долгая операция проходящая по всему проводнику и работающая с какими то файлами(работа с ними очень нагруженная)
Есть расписание которое настраивается пользователем и в следствии созданного расписания выполняется операция выше.
Проблема заключалась в том что добавилась задача на продолжение операции имела бизнес ошибку, зачем продолжать проходится по проводнику и работать с файлами, если спустя время, те места где мы работали могли обновится и иметь новые файлы/измененные?

Здесь решение было простым, до начала разработки, отказ от остановки процесса, операцию прохода прерываем навсегда и таким образом если пользователь решит продолжить, то на самом деле операция стартует с самого начала.


# После прочтения
решение с FileManager было правильным, решил что не стоило усложнять код, выделяя общую базу открытых документов в приложении являющимся службой Windows и добавляя дополнительное общение-проверку открытых файлов между этой службой и кроткосрочными процессами. Добавление попыток через Nое количество времени прекрасно справляется.
Заранее отказавшись от продолжения обхода в "Отмена или продолжить" мы избегаем и ошибок в бизнес логики и в возможной запутанности кода.
примеры 2 и 3 чтото у меня пошли мимо...
# Итог
Нужно выделять не только АТД для модели мира, но и специфичный АТД для хаоса, как программа должна себя вести при узких и особенных ситуациях.
Нужно соблюдать не баланс 50% на 50%, а скорее 99% и 1%. Ведь с ростом функциональности приложения будут расти и краевые случаи, тогда и нужно понимать/чувствовать/видеть, что баланс может быть нарушен между правильной абстракцией и ее хаосом, и в таком случае уже пора поднимать абстракцию отталкиваясь от накопившегося хаоса.