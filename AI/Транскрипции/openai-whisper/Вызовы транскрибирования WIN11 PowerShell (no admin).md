
***Предаврительная настройка:***
``` mkdir .\out -ea 0 | Out-Null ```  - создание папки для выгрузки последующего текста в поддиректорию (внимание к тому, по какому адресу каталога открыт PowerShell).

``` $PROMT = "рынок, зона, таймфрейм, стакан, риск-менеджмент, стоп-лосс, тейк-профит" ``` - словарь терминов. Слова, которые точно используются в видео. Необходимо для более точного преобразования слов из речи в нужные слова в тексте.

# Обработка одного файла:

***Команда для запуска обработки одного файла:***
```
python -m whisper ".\название_файла.формат" --model large-v3 --language ru --task transcribe --device cuda --fp16 True --output_format all --output_dir .\out --initial_prompt $PROMPT
```

# Whisper large-v3 — быстрый запуск в PowerShell

> [!important] Какой движок запускается  
> Команда `python -m whisper` относится к **оригинальному OpenAI Whisper на PyTorch**, а не непосредственно к `faster-whisper` на CTranslate2. У `faster-whisper` другой Python-интерфейс через `WhisperModel`, хотя модели и общая логика транскрибации похожи. Эта памятка составлена именно под уже проверенный мной запуск:
> 
> ```powershell
> python -m whisper
> ```

---

## 1. Структура рабочей папки

```text
WoW-PvP-Transcription\
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
└── out\
```

- `initial_prompt.txt` — общий контекст для всех трёх видео.
    
- Три остальных `.txt` — индивидуальный контекст конкретного видео.
    
- `.webm` и `.mkv` переименовывать в `.mp4` не нужно.
    
- Whisper передаёт медиафайл в FFmpeg, поэтому главное — наличие корректной аудиодорожки и установленного FFmpeg.
    

---

# 2. Как открыть PowerShell в нужной папке

## Способ 1 — через Проводник

1. Открыть рабочую папку в Проводнике.
    
2. Щёлкнуть по адресной строке.
    
3. Ввести:
    

```text
powershell
```

4. Нажать `Enter`.
    

PowerShell откроется сразу в текущей папке.

Также в Windows 11 можно нажать правой кнопкой мыши по свободному месту папки и выбрать:

```text
Открыть в терминале
```

---

## Способ 2 — перейти из уже открытого PowerShell

Полная команда:

```powershell
Set-Location -LiteralPath "D:\WoW-PvP-Transcription"
```

Короткий эквивалент:

```powershell
cd "D:\WoW-PvP-Transcription"
```

Путь с пробелами обязательно заключать в кавычки:

```powershell
cd "D:\Видео и транскрибации\WoW PvP"
```

---

## Полезные команды навигации

Текущая папка:

```powershell
Get-Location
```

Короткий вариант:

```powershell
pwd
```

Показать содержимое папки:

```powershell
Get-ChildItem
```

Короткий вариант:

```powershell
ls
```

Перейти на один уровень выше:

```powershell
cd ..
```

Перейти в подпапку:

```powershell
cd ".\out"
```

Очистить окно PowerShell:

```powershell
cls
```

---

# 3. Быстрая проверка перед запуском

Проверить Python:

```powershell
python --version
```

Проверить Whisper:

```powershell
python -m whisper --help
```

Проверить FFmpeg:

```powershell
ffmpeg -version
```

Проверить видеокарту и драйвер:

```powershell
nvidia-smi
```

Проверить доступность CUDA в PyTorch:

```powershell
python -c "import torch; print('CUDA:', torch.cuda.is_available()); print('GPU:', torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'не обнаружена')"
```

Нормальный результат:

```text
CUDA: True
GPU: NVIDIA GeForce RTX 3060 Laptop GPU
```

---

# 4. Создание папки результатов

```powershell
New-Item -ItemType Directory -Path ".\out" -Force | Out-Null
```

Короткий вариант:

```powershell
mkdir ".\out" -ErrorAction SilentlyContinue | Out-Null
```

---

# 5. Загрузка и объединение промптов

`--initial_prompt` принимает **сам текст**, а не путь к текстовому файлу.

Неправильно:

```powershell
--initial_prompt ".\initial_prompt.txt"
```

В этом случае Whisper получит только буквальный текст:

```text
.\initial_prompt.txt
```

Нужно сначала прочитать общий и индивидуальный промпты, а затем объединить их.

Выполнить один раз в текущем окне PowerShell:

```powershell
function Get-WhisperPrompt {
    param(
        [Parameter(Mandatory = $true)]
        [string]$SpecificPromptFile
    )

    $basePrompt = Get-Content `
        -LiteralPath ".\initial_prompt.txt" `
        -Raw `
        -Encoding UTF8

    $specificPrompt = Get-Content `
        -LiteralPath $SpecificPromptFile `
        -Raw `
        -Encoding UTF8

    return $basePrompt.Trim() + "`n`n" + $specificPrompt.Trim()
}
```

Пример загрузки промпта Hydramist:

```powershell
$PROMPT = Get-WhisperPrompt ".\01_hydramist_prompt.txt"
```

Посмотреть получившийся текст:

```powershell
$PROMPT
```

Проверить длину промпта:

```powershell
$PROMPT.Length
```

> [!note]  
> Использовать переменную именно `$PROMPT`, а не `$PROMT`. Это разные имена переменных.

---

# 6. Форма и параметры вызова Whisper

## Общая форма команды

```powershell
python -m whisper `
    "<путь_к_видео>" `
    --model <модель> `
    --language <язык> `
    --task <задача> `
    --device <устройство> `
    --fp16 <True/False> `
    --output_format <формат> `
    --output_dir "<папка>" `
    --word_timestamps <True/False> `
    --initial_prompt "$PROMPT"
```

Символ обратного апострофа:

```text
`
```

переносит команду на следующую строку в PowerShell.

После него не должно быть пробелов или комментариев. Иначе перенос может не сработать.

Команду можно записать и в одну строку:

```powershell
python -m whisper ".\video.mkv" --model large-v3 --language en --task transcribe --device cuda --fp16 True --output_format all --output_dir ".\out" --word_timestamps True --initial_prompt "$PROMPT"
```

---

## Значение основных параметров

|Параметр|Значение|Функция|
|---|---|---|
|`python -m whisper`|—|Запускает модуль OpenAI Whisper|
|`".\video.mkv"`|путь|Входной видео- или аудиофайл|
|`--model large-v3`|модель|Использует точную, но тяжёлую модель `large-v3`|
|`--language en`|язык|Указывает, что речь в видео английская|
|`--task transcribe`|задача|Оставляет транскрипт на исходном языке|
|`--device cuda`|устройство|Запускает вычисления на NVIDIA GPU|
|`--fp16 True`|точность|Использует FP16 на видеокарте|
|`--output_format all`|формат|Сохраняет все доступные форматы результатов|
|`--output_dir ".\out"`|папка|Указывает папку выгрузки|
|`--word_timestamps True`|таймкоды|Добавляет временные данные отдельных слов|
|`--initial_prompt "$PROMPT"`|контекст|Передаёт WoW-термины и контекст конкретного видео|

OpenAI Whisper поддерживает `initial_prompt`, пословные таймкоды, выбор языка, задачи, папки и формата вывода через CLI.

---

## Важные значения

Для английских WoW-гайдов:

```powershell
--language en
```

Не использовать:

```powershell
--language ru
```

Для английского транскрипта:

```powershell
--task transcribe
```

Не использовать `translate`, поскольку эта задача переводит речь в английский, а не создаёт русскую версию текста.

---

# 7. Запуск одного видео

## Hydramist

Сначала сформировать промпт:

```powershell
$PROMPT = Get-WhisperPrompt ".\01_hydramist_prompt.txt"
```

Затем запустить:

```powershell
python -m whisper `
    ".\01_hydramist_macros_keybinds.webm" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out" `
    --word_timestamps True `
    --initial_prompt "$PROMPT"
```

---

## Cdew — Keybindings

```powershell
$PROMPT = Get-WhisperPrompt ".\02_cdew_keybindings_prompt.txt"

python -m whisper `
    ".\02_cdew_keybindings.mkv" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out" `
    --word_timestamps True `
    --initial_prompt "$PROMPT"
```

---

## Cdew — Arena Movement

```powershell
$PROMPT = Get-WhisperPrompt ".\03_cdew_movement_prompt.txt"

python -m whisper `
    ".\03_cdew_arena_movement.mkv" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out" `
    --word_timestamps True `
    --initial_prompt "$PROMPT"
```

---

# 8. Автоматический запуск всех трёх видео

Скопировать весь блок в PowerShell:

```powershell
New-Item -ItemType Directory -Path ".\out" -Force | Out-Null

function Get-WhisperPrompt {
    param(
        [Parameter(Mandatory = $true)]
        [string]$SpecificPromptFile
    )

    $basePrompt = Get-Content `
        -LiteralPath ".\initial_prompt.txt" `
        -Raw `
        -Encoding UTF8

    $specificPrompt = Get-Content `
        -LiteralPath $SpecificPromptFile `
        -Raw `
        -Encoding UTF8

    return $basePrompt.Trim() + "`n`n" + $specificPrompt.Trim()
}

$jobs = @(
    [PSCustomObject]@{
        Video  = ".\01_hydramist_macros_keybinds.webm"
        Prompt = ".\01_hydramist_prompt.txt"
    },
    [PSCustomObject]@{
        Video  = ".\02_cdew_keybindings.mkv"
        Prompt = ".\02_cdew_keybindings_prompt.txt"
    },
    [PSCustomObject]@{
        Video  = ".\03_cdew_arena_movement.mkv"
        Prompt = ".\03_cdew_movement_prompt.txt"
    }
)

foreach ($job in $jobs) {
    if (-not (Test-Path -LiteralPath $job.Video)) {
        throw "Не найден видеофайл: $($job.Video)"
    }

    if (-not (Test-Path -LiteralPath $job.Prompt)) {
        throw "Не найден файл промпта: $($job.Prompt)"
    }

    $PROMPT = Get-WhisperPrompt $job.Prompt

    Write-Host ""
    Write-Host "============================================="
    Write-Host "Видео:  $($job.Video)"
    Write-Host "Промпт: $($job.Prompt)"
    Write-Host "============================================="
    Write-Host ""

    python -m whisper `
        $job.Video `
        --model large-v3 `
        --language en `
        --task transcribe `
        --device cuda `
        --fp16 True `
        --output_format all `
        --output_dir ".\out" `
        --word_timestamps True `
        --initial_prompt "$PROMPT"

    if ($LASTEXITCODE -ne 0) {
        throw "Whisper завершился с ошибкой при обработке: $($job.Video)"
    }

    Write-Host ""
    Write-Host "Готово: $($job.Video)"
}

Write-Host ""
Write-Host "Все три видео обработаны."
```

Видео обрабатываются последовательно:

```text
Hydramist → Cdew Keybindings → Cdew Movement
```

---

# 9. Сохранение автоматического запуска в `.ps1`

Создать файл:

```text
transcribe_wow_guides.ps1
```

Вставить в него блок автоматического запуска.

Запуск из рабочей папки:

```powershell
.\transcribe_wow_guides.ps1
```

Если PowerShell блокирует выполнение локальных скриптов:

```powershell
powershell -ExecutionPolicy Bypass -File ".\transcribe_wow_guides.ps1"
```

Права администратора не нужны.

---

# 10. Проверка MKV и WebM

Показать информацию о первом файле:

```powershell
ffmpeg -hide_banner -i ".\01_hydramist_macros_keybinds.webm"
```

Для MKV:

```powershell
ffmpeg -hide_banner -i ".\02_cdew_keybindings.mkv"
```

Нужно найти строку с аудиодорожкой, например:

```text
Audio: opus, 48000 Hz, stereo
```

или:

```text
Audio: aac, 44100 Hz, stereo
```

Более чистая проверка:

```powershell
ffprobe `
    -v error `
    -select_streams a:0 `
    -show_entries stream=codec_name,sample_rate,channels `
    -of default=noprint_wrappers=1 `
    ".\01_hydramist_macros_keybinds.webm"
```

Ожидаемый результат:

```text
codec_name=opus
sample_rate=48000
channels=2
```

---

# 11. Дополнительные режимы

## Если Whisper повторяет фразы

Повторить проблемный файл с:

```powershell
--condition_on_previous_text False
```

Пример:

```powershell
python -m whisper `
    ".\03_cdew_arena_movement.mkv" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out_retry" `
    --word_timestamps True `
    --condition_on_previous_text False `
    --initial_prompt "$PROMPT"
```

При отключении этого параметра Whisper перестаёт передавать предыдущий распознанный текст в следующее окно. Это может уменьшить зацикливание, но иногда ослабляет связность соседних фрагментов.

---

## Если термины хуже распознаются ближе к концу

Попробовать:

```powershell
--carry_initial_prompt True
```

Пример:

```powershell
python -m whisper `
    ".\01_hydramist_macros_keybinds.webm" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out_carry" `
    --word_timestamps True `
    --carry_initial_prompt True `
    --initial_prompt "$PROMPT"
```

`carry_initial_prompt` добавляет исходный промпт к каждому внутреннему окну распознавания, а не только к началу транскрибации.

Не включать оба дополнительных режима заранее. Сначала провести обычный запуск и проверить результат.

---

# 12. Основные ошибки

## Пустой промпт

Проверить:

```powershell
$PROMPT.Length
```

Если результат `0`, промпт не загрузился.

---

## Файл не найден

Проверить наличие файла:

```powershell
Test-Path ".\02_cdew_keybindings.mkv"
```

Результат должен быть:

```text
True
```

---

## Whisper не найден

```text
No module named whisper
```

Проверить установку:

```powershell
pip show openai-whisper
```

---

## FFmpeg не найден

```text
ffmpeg is not recognized
```

Проверить:

```powershell
ffmpeg -version
```

Whisper требует установленный и доступный через `PATH` FFmpeg.

---

## CUDA не работает

Проверить:

```powershell
python -c "import torch; print(torch.cuda.is_available())"
```

Если вывод:

```text
False
```

Whisper не сможет нормально использовать:

```powershell
--device cuda
```

---

## Недостаточно видеопамяти

Возможный текст ошибки:

```text
CUDA out of memory
```

Закрыть программы, использующие GPU:

- браузеры с тяжёлыми вкладками;
    
- игры;
    
- программы генерации изображений;
    
- видеоредакторы;
    
- другие локальные ИИ-модели.
    

После этого повторить запуск.

---

# 13. Минимальная памятка на 30 секунд

```powershell
cd "D:\WoW-PvP-Transcription"

mkdir ".\out" -ErrorAction SilentlyContinue | Out-Null

$BASE = Get-Content ".\initial_prompt.txt" -Raw -Encoding UTF8
$SPECIFIC = Get-Content ".\01_hydramist_prompt.txt" -Raw -Encoding UTF8
$PROMPT = $BASE.Trim() + "`n`n" + $SPECIFIC.Trim()

python -m whisper `
    ".\01_hydramist_macros_keybinds.webm" `
    --model large-v3 `
    --language en `
    --task transcribe `
    --device cuda `
    --fp16 True `
    --output_format all `
    --output_dir ".\out" `
    --word_timestamps True `
    --initial_prompt "$PROMPT"
```

---

# 14. Логика рабочего процесса

```text
1. Открыть PowerShell в папке проекта.
2. Проверить нужные видео и файлы промптов.
3. Создать папку out.
4. Объединить initial_prompt.txt с промптом видео.
5. Передать объединённый текст через --initial_prompt.
6. Запустить large-v3 на CUDA с language=en.
7. Дождаться создания файлов в out.
8. Проверить TXT, SRT и JSON.
9. Только при проблемах использовать дополнительные параметры.
```