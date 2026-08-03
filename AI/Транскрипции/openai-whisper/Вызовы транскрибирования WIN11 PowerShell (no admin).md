
***Предаврительная настройка:***
``` mkdir .\out -ea 0 | Out-Null ```  - создание папки для выгрузки последующего текста в поддиректорию (внимание к тому, по какому адресу каталога открыт PowerShell).

``` $PROMT = "рынок, зона, таймфрейм, стакан, риск-менеджмент, стоп-лосс, тейк-профит" ``` - словарь терминов. Слова, которые точно используются в видео. Необходимо для более точного преобразования слов из речи в нужные слова в тексте.

# Обработка одного файла:

***Команда для запуска обработки одного файла:***
```
python -m whisper ".\название_файла.формат" --model large-v3 --language ru --task transcribe --device cuda --fp16 True --output_format all --output_dir .\out --initial_prompt $PROMPT
```

# Whisper large-v3 — надёжный запуск в Windows PowerShell

## Главное правило

> [!danger] Не вставлять большой скрипт прямо в PowerShell  
> Многострочный блок может попасть в консоль как одна строка. Тогда исчезают необходимые разделители команд, свойств и конструкций PowerShell.
> 
> В консоль вставляются только короткие однострочные команды.
> 
> Основной код сохраняется в:
> 
> ```text
> transcribe_wow_guides.ps1
> ```

---

# 1. Структура папки

```text
D:\WOW Builds\Guides\
│
├── initial_prompt.txt
├── 01_hydramist_prompt.txt
├── 02_cdew_keybindings_prompt.txt
├── 03_cdew_movement_prompt.txt
│
├── 01_hydramist_macros_keybinds.webm
├── 02_cdew_keybindings.mkv
├── 03_cdew_arena_movement.mkv
│
└── transcribe_wow_guides.ps1
```

Папку `out` заранее создавать необязательно — скрипт создаст её сам.

---

# 2. Открытие PowerShell в нужной папке

## Через Проводник Windows 11

Открыть папку:

```text
D:\WOW Builds\Guides
```

Затем:

1. Щёлкнуть правой кнопкой мыши по пустому месту.
    
2. Выбрать **Открыть в терминале**.
    

Либо щёлкнуть по адресной строке Проводника, написать:

```text
powershell
```

и нажать `Enter`.

---

## Из уже открытого PowerShell

Вставить одной строкой:

```powershell
cd "D:\WOW Builds\Guides"
```

Проверить текущую папку:

```powershell
pwd
```

Показать файлы:

```powershell
ls
```

Путь заключён в кавычки, потому что содержит пробелы.

---

# 3. Быстрая проверка окружения

Каждая команда вставляется отдельно.

Проверка Python:

```powershell
python --version
```

Проверка Whisper:

```powershell
python -m whisper --help
```

Проверка FFmpeg:

```powershell
ffmpeg -version
```

Проверка видеокарты:

```powershell
nvidia-smi
```

Проверка CUDA в PyTorch:

```powershell
python -c "import torch; print('CUDA:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'not detected')"
```

Ожидаемый результат:

```text
CUDA: True
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
```

---

# 4. Создание файла скрипта

Находясь в рабочей папке, выполнить одной строкой:

```powershell
notepad ".\transcribe_wow_guides.ps1"
```

Если Блокнот спросит, создать ли новый файл, согласиться.

Вставить в Блокнот код из следующего раздела.

> [!important]  
> Этот код вставляется в Блокнот или другой текстовый редактор, а не непосредственно в окно PowerShell.

---

# 5. Содержимое `transcribe_wow_guides.ps1`

Скрипт специально не содержит русских букв. Поэтому он не должен ломаться даже при неудачно выбранной кодировке.

```powershell
$ErrorActionPreference = "Stop"

Set-Location -LiteralPath $PSScriptRoot

New-Item -ItemType Directory -Path ".\out" -Force | Out-Null

function Get-WhisperPrompt {
    param(
        [Parameter(Mandatory = $true)]
        [string]$SpecificPromptFile
    )

    $BasePromptFile = ".\initial_prompt.txt"

    if (-not (Test-Path -LiteralPath $BasePromptFile)) {
        throw "Base prompt file not found: $BasePromptFile"
    }

    if (-not (Test-Path -LiteralPath $SpecificPromptFile)) {
        throw "Specific prompt file not found: $SpecificPromptFile"
    }

    $basePrompt = Get-Content -LiteralPath $BasePromptFile -Raw -Encoding UTF8
    $specificPrompt = Get-Content -LiteralPath $SpecificPromptFile -Raw -Encoding UTF8

    return $basePrompt.Trim() + "`n`n" + $specificPrompt.Trim()
}

$jobs = @(
    [PSCustomObject]@{
        Video = ".\01_hydramist_macros_keybinds.webm"
        Prompt = ".\01_hydramist_prompt.txt"
    }

    [PSCustomObject]@{
        Video = ".\02_cdew_keybindings.mkv"
        Prompt = ".\02_cdew_keybindings_prompt.txt"
    }

    [PSCustomObject]@{
        Video = ".\03_cdew_arena_movement.mkv"
        Prompt = ".\03_cdew_movement_prompt.txt"
    }
)

foreach ($job in $jobs) {
    if (-not (Test-Path -LiteralPath $job.Video)) {
        throw "Video file not found: $($job.Video)"
    }

    $PROMPT = Get-WhisperPrompt -SpecificPromptFile $job.Prompt

    Write-Host ""
    Write-Host "===================================================="
    Write-Host "Starting video: $($job.Video)"
    Write-Host "Prompt file:    $($job.Prompt)"
    Write-Host "Prompt length:  $($PROMPT.Length) characters"
    Write-Host "===================================================="
    Write-Host ""

    $whisperArgs = @(
        "-m"
        "whisper"
        $job.Video
        "--model"
        "large-v3"
        "--language"
        "en"
        "--task"
        "transcribe"
        "--device"
        "cuda"
        "--fp16"
        "True"
        "--output_format"
        "all"
        "--output_dir"
        ".\out"
        "--word_timestamps"
        "True"
        "--initial_prompt"
        $PROMPT
    )

    & python @whisperArgs

    if ($LASTEXITCODE -ne 0) {
        throw "Whisper failed while processing: $($job.Video)"
    }

    Write-Host ""
    Write-Host "Completed: $($job.Video)"
}

Write-Host ""
Write-Host "===================================================="
Write-Host "All three videos have been processed."
Write-Host "Output directory: $PSScriptRoot\out"
Write-Host "===================================================="
```

В вызове Whisper аргументы помещены в массив `$whisperArgs`. Поэтому здесь не используются хрупкие обратные апострофы для переноса длинной команды.

---

# 6. Сохранение скрипта

В Блокноте:

```text
Файл → Сохранить
```

Проверить, что файл называется:

```text
transcribe_wow_guides.ps1
```

а не:

```text
transcribe_wow_guides.ps1.txt
```

Если используется окно **Сохранить как**, выбрать:

```text
Тип файла: Все файлы
Кодировка: UTF-8 с BOM
```

Поскольку сам код ASCII, он также будет корректно работать при сохранении как ANSI или обычный UTF-8.

Файлы промптов могут оставаться в UTF-8: скрипт читает их с явным параметром:

```powershell
-Encoding UTF8
```

---

# 7. Запуск скрипта

В PowerShell вставить **одну строку**:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File ".\transcribe_wow_guides.ps1"
```

Права администратора не нужны.

`-ExecutionPolicy Bypass` применяется к запускаемому процессу PowerShell и позволяет выполнить локальный скрипт без постоянного изменения системной политики.

---

## Запуск из любой папки

Можно указать полный путь:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "D:\WOW Builds\Guides\transcribe_wow_guides.ps1"
```

В начале скрипта есть:

```powershell
Set-Location -LiteralPath $PSScriptRoot
```

Поэтому скрипт самостоятельно переходит в папку, где находится сам `.ps1`, и находит расположенные рядом видео и промпты.

---

# 8. Что должно появиться после запуска

В консоли:

```text
Starting video: .\01_hydramist_macros_keybinds.webm
Prompt file:    .\01_hydramist_prompt.txt
Prompt length:  ... characters
```

После первого видео автоматически начнётся второе, затем третье.

Результаты сохранятся в:

```text
D:\WOW Builds\Guides\out
```

---

# 9. Форма и параметры вызова Whisper

Фактически скрипт выполняет такую команду:

```text
python -m whisper <video> <parameters>
```

Основные параметры:

|Параметр|Значение|Назначение|
|---|--:|---|
|`-m whisper`|—|Запускает установленный модуль OpenAI Whisper|
|`--model large-v3`|`large-v3`|Выбирает модель распознавания|
|`--language en`|английский|Указывает язык речи|
|`--task transcribe`|транскрибация|Сохраняет текст на исходном языке|
|`--device cuda`|NVIDIA GPU|Использует видеокарту|
|`--fp16 True`|FP16|Уменьшает нагрузку и ускоряет расчёты на GPU|
|`--output_format all`|все форматы|Создаёт доступные текстовые форматы результата|
|`--output_dir .\out`|папка|Указывает место сохранения|
|`--word_timestamps True`|включено|Добавляет пословные временные данные|
|`--initial_prompt`|текст|Передаёт контекст и WoW-терминологию|

`initial_prompt` и `word_timestamps` являются параметрами транскрибации OpenAI Whisper.

---

# 10. Как формируется промпт

Для каждого видео скрипт соединяет:

```text
initial_prompt.txt
```

и индивидуальный файл:

```text
01_hydramist_prompt.txt
```

или:

```text
02_cdew_keybindings_prompt.txt
```

или:

```text
03_cdew_movement_prompt.txt
```

Логика:

```powershell
$PROMPT = $basePrompt.Trim() + "`n`n" + $specificPrompt.Trim()
```

В Whisper передаётся уже готовый объединённый текст, а не пути к `.txt`-файлам.

---

# 11. Запуск только одного видео без `.ps1`

Для разового запуска можно использовать одну длинную, но **однострочную** команду.

## Hydramist

```powershell
$BASE = Get-Content ".\initial_prompt.txt" -Raw -Encoding UTF8; $SPECIFIC = Get-Content ".\01_hydramist_prompt.txt" -Raw -Encoding UTF8; $PROMPT = $BASE.Trim() + "`n`n" + $SPECIFIC.Trim(); python -m whisper ".\01_hydramist_macros_keybinds.webm" --model large-v3 --language en --task transcribe --device cuda --fp16 True --output_format all --output_dir ".\out" --word_timestamps True --initial_prompt "$PROMPT"
```

## Cdew — Keybindings

```powershell
$BASE = Get-Content ".\initial_prompt.txt" -Raw -Encoding UTF8; $SPECIFIC = Get-Content ".\02_cdew_keybindings_prompt.txt" -Raw -Encoding UTF8; $PROMPT = $BASE.Trim() + "`n`n" + $SPECIFIC.Trim(); python -m whisper ".\02_cdew_keybindings.mkv" --model large-v3 --language en --task transcribe --device cuda --fp16 True --output_format all --output_dir ".\out" --word_timestamps True --initial_prompt "$PROMPT"
```

## Cdew — Movement

```powershell
$BASE = Get-Content ".\initial_prompt.txt" -Raw -Encoding UTF8; $SPECIFIC = Get-Content ".\03_cdew_movement_prompt.txt" -Raw -Encoding UTF8; $PROMPT = $BASE.Trim() + "`n`n" + $SPECIFIC.Trim(); python -m whisper ".\03_cdew_arena_movement.mkv" --model large-v3 --language en --task transcribe --device cuda --fp16 True --output_format all --output_dir ".\out" --word_timestamps True --initial_prompt "$PROMPT"
```

В этих командах все отдельные инструкции разделены точками с запятой. Поэтому команда остаётся синтаксически корректной, даже когда вставляется в PowerShell одной строкой.

Для трёх видео подряд предпочтительнее `.ps1`.

---

# 12. Если скрипт не запускается

## Проверить расширение

```powershell
Get-ChildItem ".\transcribe_wow_guides*"
```

Должно быть:

```text
transcribe_wow_guides.ps1
```

Не должно быть:

```text
transcribe_wow_guides.ps1.txt
```

---

## Проверить синтаксис без запуска Whisper

```powershell
$errors = $null; [System.Management.Automation.Language.Parser]::ParseFile((Resolve-Path ".\transcribe_wow_guides.ps1"), [ref]$null, [ref]$errors) | Out-Null; $errors
```

Если ничего не выведено, синтаксических ошибок не обнаружено.

---

## Проверить наличие всех файлов

Вставить одной строкой:

```powershell
@(".\initial_prompt.txt", ".\01_hydramist_prompt.txt", ".\02_cdew_keybindings_prompt.txt", ".\03_cdew_movement_prompt.txt", ".\01_hydramist_macros_keybinds.webm", ".\02_cdew_keybindings.mkv", ".\03_cdew_arena_movement.mkv") | ForEach-Object { "{0,-55} {1}" -f $_, (Test-Path -LiteralPath $_) }
```

Для каждого файла должно быть:

```text
True
```

---

# 13. Краткое повторение

```text
1. Видео, промпты и .ps1 лежат в одной папке.
2. Большой скрипт не вставляется в консоль.
3. Скрипт сохраняется как transcribe_wow_guides.ps1.
4. В PowerShell вставляется только однострочная команда запуска.
5. Скрипт сам переходит в свою папку.
6. Скрипт соединяет общий и индивидуальный промпты.
7. Три видео обрабатываются последовательно.
8. Результаты сохраняются в папку out.
```

Команда запуска:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "D:\WOW Builds\Guides\transcribe_wow_guides.ps1"
```