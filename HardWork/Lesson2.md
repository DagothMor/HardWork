# Отчет
Трудности возникали максимальные, поскольку мой проект на .net, а сам я пользуюсь исключительно windows.
https://mijailovic.net/2019/01/03/sharpfuzz/
```
Unfortunately, it never got popular enough in the context of managed languages such as C# or Java. One of the reasons is that fuzzers are usually security-oriented, and as such they are often used with targets written in C/C++ to find memory-corruption vulnerabilities, which is a class of problems that is completely eliminated in managed programming languages.
```
Попытался использовать движок libfuzzer, в статье говорилось о том что запускается на Linux и windows
https://github.com/Metalnem/sharpfuzz/blob/master/docs/libFuzzer.md
Однако почитав комментарий к исходному коду, огорчился
```
/// <summary>
        /// LibFuzzer class contains the libFuzzer runner. It enables users
        /// to fuzz their code with libFuzzer by using the libFuzzer-dotnet
        /// binary, which acts as a bridge between the libFuzzer and the
        /// managed code (it currently works only on Linux).
        /// </summary>
        public static class LibFuzzer
```
Попробовал использовать Fuzzlin
https://github.com/jakobbotsch/Fuzzlyn
Используя следующую команду смог запустить 
```
dotnet fuzzlyn.dll --host "C:\Users\siegf\source\repos\FizzNetCoreTest\FizzNetCoreTest\bin\Debug\net7.0\FizzNetCoreTest.exe" --num-programs 1000000 --parallelism -1 --output-events-to "C:\Users\siegf\source\repos\FizzNetCoreTest\FizzNetCoreTest\testcases\res.txt"
```
Результат неутешительный, приходили seed которые генерировали одинаковый код 
```
{"Kind":"ExampleFound","Timestamp":"2023-07-01T23:35:38.1677325+00:00","Example":{"Seed":13577392617209568682,"Kind":"Crash","Message":""},"RunSummary":null}
```
Воспроизводился текст программы таким образом
```
dotnet fuzzlyn.dll --host "C:\Users\siegf\source\repos\FizzNetCoreTest\FizzNetCoreTest\bin\Debug\net7.0\FizzNetCoreTest.exe" --seed 1062009786281114503 --reduce
```

Таким образом пришлось вернуться к sharpfuzzer. Использовал WSL, скачал с microsoft store ubuntu 20.04,скачал все нужные библиотеки 
.Net
```
sudo apt install dotnet-sdk
```
Samba для создания диска который будет посредником между моей windows и машиной Ubuntu
```
sudo apt-get install samba
```
```
sudo nano /etc/samba/smb.conf
[global]
workgroup = WORKGROUP
wins support = yes

[SharedFolder]
   path = /home/aboba/shared
   browsable = yes
   guest ok = yes
   read only = no
   create mask = 0777
   directory mask = 0777
   force user = aboba
   
```

```
sudo service smbd restart
```
Режим Qemu QEMU (Quick Emulator) — это программное обеспечение для эмуляции процессоров и виртуальных машин.
```
sudo apt-get install qemu
```
В итоге я смог запустить afl для dotnet
```
afl-fuzz -Q -i testcases -o Findings -- ./bin/Debug/net7.0/linux-x64/publish/FizzNetCoreTest.dll
```
количество строк в моем проекте 200. Отловил null при передаче параметров, больше увы ничего, поскольку в большей части логики происходит работа с редактором реестра windows