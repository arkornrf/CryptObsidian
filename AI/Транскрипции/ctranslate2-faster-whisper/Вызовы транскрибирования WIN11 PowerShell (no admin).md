
# Предаврительная настройка:
```
mkdir .\out -ea 0 | Out-Null
```
- создание папки для выгрузки последующего текста в поддиректорию (внимание к тому, по какому адресу каталога открыт PowerShell).

---


``` 
$PROMPT = @"
Контекст: русскоязычная лекция/курс по трейдингу криптовалют (спот/фьючерсы, маржа, риск-менеджмент, стратегии входа/выхода, работа с уровнями, психология, статистика сделок).

Язык и стиль:
• Язык транскрибации — русский. Английские термины/тикеры — в ОРИГИНАЛЕ и ВЕРХНЕМ регистре (BTC, ETH, BTCUSDT, PERP).
• Сохраняй техническую лексику, числа и знаки: десятичная точка (1.25), проценты без пробела (2.5%), доллар как $; плечо «x5».
• Убирай «мусорные» междометия («э-э», «ну», «как бы») и многократные повторы («и так далее» → «и т. д.» один раз).
• Не вставляй случайные иностранные слова без контекста (напр. «był», «jest»). Если это слышится как «был/есть» — пиши по-русски.

Нормализация терминов (пиши ровно так):
• «вынос» (в т. ч. «вынос стопов», «сбор ликвидности»), «пробой», «ложный пробой», «ретест», «наклонка» (наклонная линия тренда), «уровень», «диапазон».
• «скринер» (не «скриннер»), «лонг», «шорт», «шортисты», «быки», «медведи», «сквиз», «памп/дамп», «флет», «волатильность», «ликвидность», «спред», «проскальзывание».
• Типы ордеров/режимы: market, limit, stop, stop-limit, OCO, post-only, reduce-only, isolated/cross margin, «x3», «x10».
• Сокращения: SL (стоп-лосс), TP (тейк-профит), BE (безубыток), R, R:R (напр. «R:R 1:3»), PnL, ROI, OI (open interest), CVD, DOM/LOB, VWAP, ATR, EMA(21/50/200), SMA, MACD, RSI, BB, Ichimoku, OB (order block), FVG (imbalance), S&D, PoC, VAH/VAL, TPO, Fair Value Gap.
• Паттерны: «голова и плечи», «двойная вершина/дно», «клин», «флаг», «пробой/ложный пробой», «ретест», «диапазон», «канал», «трендовая линия».

Биржи/платформы (пиши как названия): Binance, Bybit, OKX, Gate, Bitget, MEXC, KuCoin, Deribit, BitMEX.

Пары/контракты (пример): BTCUSDT, ETHUSDT, SOLUSDT, TONUSDT, ARBUSDT, AVAXUSDT, PEPEUSDT, WIFUSDT; BTC-PERP, ETH-PERP; XBTUSD.

Алиасы → стандартные формы тикеров:
• «биток»/«биткоин» → BTC; «эфир» → ETH; «сола/солана» → SOL; «тон» → TON; «бинанс коин/бнб» → BNB; «лайткоин» → LTC; «полкадот» → DOT; «нир» → NEAR; «атом/космос» → ATOM; «х-р-п» → XRP; «монеро» → XMR.
• «арб/арбитрум» → ARB; «авакс/аваланч» → AVAX; «шиба» → SHIB; «рендер» → RNDR; «селестия/тия» → TIA; «тао/байтенсор» → TAO; «ондо» → ONDO; «лейер зеро/зиро» → ZRO; «стрк/старкнет» → STRK; «зк/зедкей» → ZK.

Расширенный список тикеров (фиксируй ВЕРХНИМ РЕГИСТРОМ): 
• L1/L2/блю-чипы: BTC, ETH, SOL, BNB, XRP, ADA, DOGE, TON, TRX, DOT, AVAX, MATIC/POL, OP, ARB, SEI, SUI, APT, NEAR, ATOM, ICP, EGLD, XLM, XTZ, LTC, BCH.
• DeFi/деривативы/DEX: UNI, AAVE, SUSHI, CRV, CVX, BAL, GMX, DYDX, JOE, RAY, ORCA, CAKE, 1INCH, SNX, LDO, RPL, RDNT.
• Инфра/хранилища/индексы: GRT, FIL, AR, STORJ, BTT, KAS.
• AI/DePIN/RWA/идеи: ASI, RNDR, TAO, WLD, FET, AGIX, OCEAN, HNT, IOTX, IOTA, NODL, ONDO, POLYX.
• Мем-сегмент (без склонений): SHIB, PEPE, WIF, BONK, FLOKI, MEW, DOGE.

Правила оформления текста:
• Завершай предложения точкой. Длинные фразы дели на короткие смысловые предложения.
• Не повторяй одно слово/фразу более 2 раз подряд; «и т. д.» — максимум один раз в предложении.
• Числа и символы: «Funding 0.01%/8h», «PnL +2.5%», «EMA200 на H4», «R:R 1:3», «$100».
• Вопросы/перечни оформляй знаками «?», «:» и «;» (например: «Где пробивало: уровень D1; ретест H4; точка входа — после закрепления.»).

Цель: точная техническая транскрибация (без домыслов), с сохранением профильной терминологии, тикеров и чисел. Если термин не распознан — лучше оставить англ. аббревиатуру/тикер, чем придумывать русскую форму.
"@
```
- слова, которые точно используются в видео. Необходимо для более точного преобразования слов из речи в нужные слова в тексте.

```
$MODEL_DIR = "D:\AI\CTranslate2_Faster-whisper_large_v3\models\faster-whisper-large-v3" 
```
- адрес директории с файлом llvm "model.bin".
$MODEL_DIR = "D:\Trading\Baga\models\faster-whisper-large-v3"

# Обработка одного файла:

***Вызов для запуска обработки одного файла:***
```
whisper-ctranslate2 ".\название_файла.формат" `
--model_directory "$MODEL_DIR" `
--device cuda --compute_type float16 `
--language ru --task transcribe `
--output_dir .\out --output_format all `
--vad_filter True `
--initial_prompt $PROMPT
```

***==ФАЙНТЮН== Команда для запуска обработки одного файла:***
```
whisper-ctranslate2 ".\название_файла.формат" `
  --model_directory "$MODEL_DIR" `
  --device cuda --compute_type float16 `
  --language ru --task transcribe `
  --vad_filter True `
  --vad_threshold 0.6 `
  --vad_min_speech_duration_ms 500 `
  --vad_max_speech_duration_s 30 `
  --beam_size 5 --patience 1 `
  --length_penalty 1.0 `
  --repetition_penalty 1.1 `
  --no_repeat_ngram_size 3 `
  --suppress_blank True `
  --suppress_tokens -1 `
  --condition_on_previous_text False `
  --temperature 0 `
  --compression_ratio_threshold 2.4 `
  --logprob_threshold -0.5 `
  --no_speech_threshold 0.6 `
  --word_timestamps True `
  --initial_prompt $PROMPT `
  --output_dir .\out --output_format all
```

# Несколько файлов:

***Вызов для 1 формата (mp4):***
```
Get-ChildItem -File -Filter *.mp4 | ForEach-Object { 
  Write-Host "Обрабатываю: $($_.Name)"
  whisper-ctranslate2 "$($_.FullName)" `
    --model_directory "$MODEL_DIR" `
    --device cuda --compute_type float16 `
    --language ru --task transcribe `
    --output_dir .\out --output_format all `
    --vad_filter True `
    --initial_prompt $PROMPT
}
```

***Множество форматов:***
``` $ext = @('.wav','.mp3','.m4a','.mp4','.webm','.mkv','.mov') ``` - если нужно обработать сразу множество форматов файлов

***Вызов для обработки множества форматов:***
```
Get-ChildItem -File | Where-Object { $ext -contains $_.Extension.ToLower() } | ForEach-Object {
  Write-Host "Обрабатываю: $($_.Name)"
  whisper-ctranslate2 "$($_.FullName)" `
    --model_directory "$MODEL_DIR" `
    --device cuda --compute_type float16 `
    --language ru --task transcribe `
    --output_dir .\out --output_format all `
    --vad_filter True `
    --initial_prompt $PROMPT
}
```

***==ФАЙНТЮН== вызов для обработки одного формата:***
```
Get-ChildItem -File -Filter *.mp4 | ForEach-Object { 
  Write-Host "Обрабатываю: $($_.Name)"
  whisper-ctranslate2 "$($_.FullName)" `
    --model_directory "$MODEL_DIR" `
    --device cuda --compute_type float16 `
    --language ru --task transcribe `
    --vad_filter True `
    --vad_threshold 0.6 `
    --vad_min_speech_duration_ms 500 `
    --vad_max_speech_duration_s 30 `
    --beam_size 5 --patience 1 `
    --length_penalty 1.0 `
    --repetition_penalty 1.1 `
    --no_repeat_ngram_size 3 `
    --suppress_blank True `
    --suppress_tokens -1 `
    --condition_on_previous_text False `
    --temperature 0 `
    --compression_ratio_threshold 2.4 `
    --logprob_threshold -0.5 `
    --no_speech_threshold 0.6 `
    --word_timestamps True `
    --initial_prompt $PROMPT `
    --output_dir .\out --output_format all
}
```

