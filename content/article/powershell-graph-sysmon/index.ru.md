---
title: 'Анализ связей между приложениями с использованием SysMon'
date: 2023-12-14T22:39:10-08:00
draft: true
---

У нас есть один или несколько серверов, и мы хотим знать, как запущенные на них приложения связаны между собой. Как они взаимодействуют по сети. Хотим увидеть граф связей между приложениями.
<!--more-->

## Сценарий

У нас есть один или несколько серверов, и мы хотим знать, как запущенные на них приложения связаны между собой. Как они взаимодействуют по сети. Хотим увидеть граф связей между приложениями.

## Эксперимент

В рамках эксперимента мы будем использовать следующие инструменты:

- [Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) - иструмент для мониторинга за активностью всего. Кстати есть и [Sysmon for Linux](https://github.com/Sysinternals/SysmonForLinux)
- Конфигурационные файлы для Sysmon [отсюда](https://github.com/SwiftOnSecurity/sysmon-config). Это конфигурация по-умолчанию, для ваших целей можно сделать свою

Попросту говоря, [Sysmon](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) это системный сервис + дравер, устанавливаемый в систему. Он умеет перехватывать различные операции, происходящие в системе и сохранять эти события их в EventLog. Параметр `-i` принимает путь к конфигурационному файлу, в котором можно задавать фильтрацию того, чего вы хотите или не хотите видеть в журнале событий. Кроме всего прочего Sysmon видит сетевые подключения, которые тот или иной процесс пытается установить с удаленными системами. Таким образом мы можем узнать какой процесс, когда и куда ходил, по какому протоколу и на какие порты пытался отправлять свои пакеты. Имея эти данные с одной машины мы можем увидеть процессы и их сетевые взаимодействия. Собрав данные с нескольких машин можно увидеть связи между ними.

В этом эксперименте мы попробуем провернуть это все на локальной машине.

### Подготовка

Прежде всего установим Sysmon вмест с его конфигом. Ну, то есть, скачаем, запустим процесс `cmd` с повышенными привилегиями, чтобы система дала установить драйвер, и пеенаправим вывод в файл, чтобы увидеть, что же вернет sysmon. Поскольку он запускается с повышенными привилегиями это, наверное, единственный способ получить его вывод.

```powershell
$resFileName = "sysmonInstallResult.txt"

# качнем в TEMP по-умолчанию
Invoke-WebRequest -Uri "https://live.sysinternals.com/Sysmon.exe" -OutFile "$($env:TEMP)\sysmon.exe"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" -OutFile "$($env:TEMP)\sysmonconfig-export.xml"
```

На всякий случай поясняю что делает эта команда. Она запускает `cmd.exe`, который расположен по пути из `$env:comspec`, и передает ему примерно вот такое `/c %TEMP%\sysmon.exe -i %TEMP%\sysmonconfig-export.xml -accepteula > %TEMP%\sysmonInstallResult.txt`. Таким образом при запуске стартанет процесс `cmd.exe`, среагирует UAC, покажет вам окно и спросит разрешения, дальше запустится сам sysmon. Его вывод будет перенаправлен туда, куда мы указали, за счет оператора `>`. Затем мы этот вывод читаем и выводим, чтобы понять, сработало или нет.

```powershell
# запустим процесс с повышенными привилегиями и оператором перенаправления
Start-Process -FilePath $env:comspec  -ArgumentList ("/c", "$($env:TEMP)\sysmon.exe", "-i", "$($env:TEMP)\sysmonconfig-export.xml", "-accepteula", ">", "$($env:TEMP)\$resFileName") `
              -Verb runas

gc "$($env:TEMP)\$resFileName"
```

Теперь нужно добавить себя в группу "Event Log Readers" если вы еще не там, на всякий случай. .NET Interactive notebooks в VSCode отчего-то не работают, когда VSCode запущен с повышеными привилегиями. После этого все равно придется перелогиниться, поэтому это действие можно проделать руками.

### Игра

Если все прошло хорошо - мы готовы. 

Установим модуль, который умеет работать с графами. На данный момент доступна только alpha версия, но она, вроде, работает. Сам модуль расположен [тут](https://github.com/eosfor/PSGraph/tree/master). Ветка `dev`, из которой периодически собирается alpha [тут](https://github.com/eosfor/PSGraph/tree/dev). Версия в ветке `master` тоже рабочая, но она не умеет многое из того что нам нужно.

```powershell
Install-Module -Name PSQuickGraph -AllowPrerelease -RequiredVersion "2.0.1-alpha"
```

Импортнем его

```powershell
Import-Module PSQuickGraph -RequiredVersion "2.0.1"
```

Теперь, собственно, самое интересное. Нам нужно прочитать события из журнала событий, построить на них граф и нарисовать его, чем и займемся. Прежде всего, прочитаем журнал. Sysmon пишет события в `Microsoft-Windows-Sysmon/Operational`. Нас интересуют события с `id=3`. Это - попытки установления сетевых соединений процессами. Выгрузив их из журнала событий мы преобразум их в массив объектов, разбирая на именованные свойства, чтобы с ними было проще работать позже. Каждый объект, кроме всего прочего содержит source и destination маркеры. Source маркер это `<имя процесса>:<processId>`, а destination - `<IP адрес назначения>:<порт назначения>`. Адреса используются потому, что DNS имена не всегда известны.

```powershell
$ids = Get-WinEvent -LogName Microsoft-Windows-Sysmon/Operational | ? {$_.id -eq 3}
$commObjects = $ids | % {
    New-Object psobject -Property @{ 
        RuleName            = $_.Properties[0].value
        UtcTime             = $_.Properties[1].value
        ProcessGuid         = $_.Properties[2].value
        ProcessId           = $_.Properties[3].value
        Image               = $_.Properties[4].value
        User                = $_.Properties[5].value
        Protocol            = $_.Properties[6].value
        Initiated           = $_.Properties[7].value
        SourceIsIpv6        = $_.Properties[8].value
        SourceIp            = $_.Properties[9].value
        SourceHostname      = $_.Properties[10].value
        SourcePort          = $_.Properties[11].value
        SourcePortName      = $_.Properties[12].value
        DestinationIsIpv6   = $_.Properties[13].value
        DestinationIp       = $_.Properties[14].value
        DestinationHostname = $_.Properties[15].value
        DestinationPort     = $_.Properties[16].value
        DestinationPortName = $_.Properties[17].value
        SourceString   = "$($_.Properties[4].value)`:$($_.Properties[3].value)" # <имя процесса>:<processId>
        DestinationString   = "$($_.Properties[14].value)`:$($_.Properties[16].value)" # <IP адрес назначения>:<порт назначения>
    }
}
```

Теперь можно построить граф взаимодействий. Для этого надо сначала создать пустой граф командой `New-Graph`, а затем наполнить его. Чтобы это сделать мы просто бежим по всем объектам и добавляем в граф ребра от `source` маркера в `destination` маркер. При этом, команда умеет понять, существуют в графе соответствующие вершины или нет. Если вершин нет - они добавляются и между ними создается направленное ребро. Если вершины есть, то новые не создаются, просто добавляется еще одно ребро. Таким образом при создании графа не нужно думать добавляли мы такую вершину или нет, команда сделает все сама, просто подавайте данные.

```powershell
$g = New-Graph
$commObjects | % {
    Add-Edge -From $_.SourceString -To $_.DestinationString -Graph $g
}

$g
```

Граф собран. Покрасим некоторые вершины. Мы хотим выделить одним цветом те вершины, которые используются несколькими процессами. Другими словами, если, к одному и тому же `destination` обращаются с течением времени несколько разных `source`, то в эту вершину будет входить больше чем одно ребро. Другим цветом мы хотим покрасить вершины, у которых нет входящих ребер. Но чтобы не слишком перегружать картинку, выберем только те, у которых 0 входящих и больше двух исходящих ребер.

Для этого нам нужно просто пробежать по всем вершинам графа, для каждой из них выяснить количество входящих и исходящих ребер и задать соответствующие цвета этой вершине

```powershell
$g.Vertices | % { if ($g.InDegree($_) -gt 1) { $_.GVertexParameters.Fillcolor = [QuikGraph.Graphviz.Dot.GraphvizColor]::AntiqueWhite } }
$g.Vertices | % { if ( ($g.InDegree($_) -eq 0) -and ($g.OutDegree($_) -gt 2) ) { $_.GVertexParameters.Fillcolor = [QuikGraph.Graphviz.Dot.GraphvizColor]::BlueViolet } }
```

Ну и наконец мы можем экспортировать граф. Это можно сделать в нескольких различных форматах. Первый из них, `Graphviz`. Это, так называемый, [dot](https://www.graphviz.org/doc/info/lang.html) формат, текстовый формат используемый утилитой [graphviz](https://www.graphviz.org/) и некоторыми другими, для хранения графов. Другой формат, который нас интересует, `MSAGL_MDS`. В этом случае создается SVG визуализация средствами библиотеки [MSAGL](https://github.com/microsoft/automatic-graph-layout). Команда поддерживат еще несколько форматов, но о них в другой раз.

```powershell
Export-Graph -Graph $g -Path "$($env:TEMP)\comms.svg" -Format MSAGL_MDS
Export-Graph -Graph $g -Path "$($env:TEMP)\comms.gv" -Format Graphviz
```

Выглядит все это примерно вот так

{{< video type="youtube" id="LuRo8GEwp1w" >}}