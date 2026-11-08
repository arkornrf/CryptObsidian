---
title: "Vivaldi — доступ только через Clash и VPN"
created: 2026-11-08
tags:
  - network
  - vpn
  - windows
  - vivaldi
  - clash
status: "Проверено: без системного прокси сайты недоступны, с прокси работают"
---

# Vivaldi — доступ только через Clash и VPN

> [!success] Что уже проверено
> На моём ПК при выключенном системном прокси Vivaldi не открывает сайты, включая ipify. После включения прокси доступ восстанавливается.
> Это подтверждает проверенный сценарий; остановку ядра, отказ VPN-узла и IPv6 проверяем отдельно по чек-листу ниже.

## Быстрая памятка

| Параметр | Значение |
| --- | --- |
| ОС | Windows 11 |
| Браузер | Vivaldi, процесс `vivaldi.exe` |
| Клиент | Clash Verge Rev / ядро Mihomo |
| Режим | Rule |
| System Proxy | ON для работы браузера |
| TUN | OFF — эта инструкция рассчитана на системный прокси |
| Локальный прокси | `127.0.0.1:7897` |
| Группа маршрутизации | `VPN-ROUTE`, без DIRECT и без вложенного выхода DIRECT |
| Правило Firewall | `Vivaldi-Block-NonLoopback` |

**Ожидаемое поведение:** VPN доступен → браузер работает через него; VPN или Clash недоступен → запросы завершаются ошибкой, прямого выхода браузера нет.

## Как устроена защита

Защита состоит из двух независимых частей:

1. **Windows Firewall** запрещает указанному `vivaldi.exe` соединения с любыми адресами вне loopback. Браузер может обращаться к локальному Clash, но не напрямую к сайтам или устройствам домашней сети.
2. **Clash** отправляет запросы Vivaldi только в VPN-группу. В этой группе не должно быть DIRECT, иначе сам Clash сможет вывести трафик браузера напрямую.

Это называется **kill switch для приложения**, или **fail-closed**: при отказе связи доступ закрывается.

| Сбой | Что обеспечивает защиту |
| --- | --- |
| VPN-узел недоступен, Clash работает | VPN-маршрут без DIRECT; запрос заканчивается ошибкой |
| Clash закрыт или системный прокси выключен | Firewall блокирует внешний выход `vivaldi.exe` |
| Браузер пытается обойти прокси | Firewall блокирует его внешние соединения |
| Clash переведён в Direct | Защита от выхода через сам Clash нарушается — так не делать |

## 1. Настройки Clash

Перед изменением сохранить копию действующего YAML. Не создавать второй раздел `rules:` и не дублировать существующие ключи.

На верхнем уровне конфигурации:

```yaml
mode: rule
find-process-mode: always
```

Добавить **первым правилом существующего списка**:

```yaml
rules:
  - PROCESS-NAME,vivaldi.exe,VPN-ROUTE
  # Далее — остальные существующие правила.
```

У группы `VPN-ROUTE` проверить состав:

- Только нужные VPN-узлы и резерв DNSTT; при необходимости допустим REJECT.
- Нет DIRECT ни непосредственно, ни во вложенных группах.
- Если используется ручная группа `select`, выбран нужный VPN.
- Не переключать общий режим в Direct. В Global правила Rule не применяются — результат зависит от выбора GLOBAL.

> [!important] Российские сайты в Vivaldi
> Это правило выше исключений `.ru`, `.su`, `.рф` и направляет через VPN весь распознанный трафик Vivaldi. Другие приложения сохраняют свою маршрутизацию. Если нужно открывать российские сайты напрямую, использовать другой браузер.

После применения проверить в Connections/Logs: запрос Vivaldi должен идти через `VPN-ROUTE` и выбранный VPN-узел. Если процесс не распознаётся, он может попасть под другие правила — это нужно исправить до использования чувствительных сервисов.

## 2. Установка правила Windows Firewall

> [!note] Уже установлено
> Повторять этот раздел на текущем ПК не нужно. Он оставлен для восстановления и второго ноутбука.

Сохранить незавершённую работу. Оставить Vivaldi запущенным, чтобы команда определила путь к исполняемому файлу. Открыть **PowerShell от имени администратора**.

```powershell
$ErrorActionPreference = "Stop"

$vivaldiPaths = @(
    Get-Process -Name vivaldi -ErrorAction Stop |
    Select-Object -ExpandProperty Path -Unique |
    Where-Object { $_ }
)

if ($vivaldiPaths.Count -ne 1) {
    throw "Не удалось однозначно определить путь Vivaldi."
}

if (Get-NetFirewallProfile | Where-Object { -not $_.Enabled }) {
    throw "Один из профилей Windows Firewall выключен. Сначала включи его."
}

$ruleName = "Vivaldi-Block-NonLoopback"

if (Get-NetFirewallRule -Name $ruleName -ErrorAction SilentlyContinue) {
    throw "Такое правило уже существует. Повторно создавать его не нужно."
}

New-NetFirewallRule `
    -Name $ruleName `
    -DisplayName "Vivaldi: block direct external connections" `
    -Direction Outbound `
    -Action Block `
    -Program $vivaldiPaths[0] `
    -Protocol Any `
    -RemoteAddress @(
        "0.0.0.0-126.255.255.255",
        "128.0.0.0-255.255.255.255",
        "::2-ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"
    ) `
    -Profile Any `
    -Enabled True

Write-Host "Правило создано для: $($vivaldiPaths[0])"
```

После выполнения полностью закрыть Vivaldi и открыть заново. Перезагрузка Windows обычно не требуется.

### Что означают параметры

| Параметр | Назначение |
| --- | --- |
| `Direction Outbound` | Ограничить исходящие соединения |
| `Action Block` | Запретить совпавший трафик |
| `Program` | Применить правило только к найденному пути `vivaldi.exe` |
| `Protocol Any` | Охватить TCP, UDP и другие IP-протоколы |
| Два диапазона IPv4 | Заблокировать адреса вне `127.0.0.0/8` |
| Диапазон IPv6 | Заблокировать назначения от `::2` и выше; сохранить loopback `::1` |
| `Profile Any` | Применить к доменному, частному и публичному профилям Windows |

Правило не ограничивает локальный доступ только портом 7897: оно оставляет доступ к другим локальным сервисам. Это защита от случайного внешнего выхода браузера, а не изоляция от всех программ на ПК.

## 3. Дополнительная настройка Vivaldi

В настройках найти **WebRTC** и выключить **Broadcast IP for Best WebRTC Performance** — разрешение раскрывать IP ради работы WebRTC. Название перевода может отличаться.

Некоторые браузерные звонки могут перестать работать или работать иначе. Переключатель дополняет Firewall, но не заменяет его.

## 4. Проверка защиты

> [!tip] Сначала подготовить фиксацию результата
> До тестов открыть Connections/Logs Clash. Если нужно сохранить вывод, заранее подготовить запись экрана или экспорт журнала. После остановки Clash его журнал может стать недоступен.
> Проверять на ipify, без входа в Claude или другой аккаунт.

### Рабочий режим

- [ ] Clash запущен, Rule, System Proxy ON, TUN OFF.
- [ ] Выбран рабочий VPN-узел.
- [ ] В Vivaldi открыть `https://api.ipify.org/?check=working`.
- [ ] Получен ожидаемый IP выхода VPN.
- [ ] В Clash запрос Vivaldi идёт через VPN, не DIRECT.

### Системный прокси выключен

- [ ] Выключить System Proxy, перезапустить Vivaldi.
- [ ] Открыть `https://api.ipify.org/?check=no-proxy`.
- [ ] Ожидаемый результат — ошибка соединения, IP провайдера не появляется.
- [ ] Включить System Proxy и подтвердить восстановление доступа через VPN.

### Clash полностью остановлен

- [ ] Полностью завершить Clash и его ядро, а не только закрыть окно.
- [ ] Проверить состояние системного прокси Windows, затем перезапустить Vivaldi.
- [ ] Открыть `https://api.ipify.org/?check=clash-off`.
- [ ] Ожидаемый результат — ошибка соединения.
- [ ] Запустить Clash, включить System Proxy и повторить рабочую проверку.

### VPN недоступен при работающем Clash

Для контролируемой проверки можно временно выбрать REJECT в VPN-группе, если он туда добавлен. Это проверяет отказ маршрута, но не все возможные сетевые сбои VPN.

- [ ] Запрос в Vivaldi завершается ошибкой.
- [ ] В логах нет перехода этого запроса в DIRECT.
- [ ] После возврата рабочего VPN доступ восстанавливается.

Проверку IPv6 выполнить также в сети с реально работающим IPv6: без этого успешный тест IPv4 не является отдельным подтверждением защиты IPv6.

## 5. Команды проверки состояния

Все команды ниже выполняются в PowerShell. Для управления Firewall нужны права администратора.

### Правило включено?

```powershell
Get-NetFirewallRule -Name "Vivaldi-Block-NonLoopback" |
    Format-List Name, Enabled, Direction, Action, Profile
```

Ожидается: `Enabled=True`, `Direction=Outbound`, `Action=Block`, `Profile=Any`.

### К какому файлу привязано?

```powershell
Get-NetFirewallRule -Name "Vivaldi-Block-NonLoopback" |
    Get-NetFirewallApplicationFilter |
    Format-List Program
```

### Какие адреса блокируются?

```powershell
Get-NetFirewallRule -Name "Vivaldi-Block-NonLoopback" |
    Get-NetFirewallAddressFilter |
    Format-List RemoteAddress
```

### Работает ли Firewall?

```powershell
Get-NetFirewallProfile | Format-Table Name, Enabled
```

### Слушает ли локальный прокси порт 7897?

```powershell
Get-NetTCPConnection -State Listen -LocalPort 7897 -ErrorAction SilentlyContinue |
    Format-Table LocalAddress, LocalPort, OwningProcess
```

Наличие слушателя подтверждает только локальный порт, не работоспособность VPN.

### Какой путь у запущенного Vivaldi?

```powershell
Get-Process -Name vivaldi |
    Select-Object -ExpandProperty Path -Unique
```

## 6. После обновления или переустановки Vivaldi

Правило привязано к пути файла, а не просто к названию браузера. Если путь не изменился, пересоздавать правило не нужно. Если изменился — обновить привязку.

Открыть Vivaldi без чувствительных вкладок, затем PowerShell от администратора:

```powershell
$ErrorActionPreference = "Stop"

$vivaldiPaths = @(
    Get-Process -Name vivaldi -ErrorAction Stop |
    Select-Object -ExpandProperty Path -Unique |
    Where-Object { $_ }
)

if ($vivaldiPaths.Count -ne 1) {
    throw "Не удалось однозначно определить путь Vivaldi."
}

Get-NetFirewallRule -Name "Vivaldi-Block-NonLoopback" -ErrorAction Stop |
    Get-NetFirewallApplicationFilter |
    Set-NetFirewallApplicationFilter -Program $vivaldiPaths[0]

Write-Host "Правило привязано к: $($vivaldiPaths[0])"
```

Полностью перезапустить браузер и повторить проверку с выключенным прокси. Для второй установки Vivaldi в другом каталоге потребуется отдельное правило.

## 7. Временное отключение и откат

> [!warning] Прямой выход снова станет возможен
> Перед отключением защиты закрыть чувствительные вкладки и браузер. Эти команды не отключают VPN, а снимают запрет Firewall.

Временно выключить правило:

```powershell
Disable-NetFirewallRule -Name "Vivaldi-Block-NonLoopback"
```

Включить обратно:

```powershell
Enable-NetFirewallRule -Name "Vivaldi-Block-NonLoopback"
```

Удалить правило полностью:

```powershell
Remove-NetFirewallRule -Name "Vivaldi-Block-NonLoopback"
```

Удаление не меняет YAML Clash. Для полного возврата прежней маршрутизации отдельно восстановить сохранённый YAML или прежний порядок правил.

## Ограничения и типичные проблемы

| Симптом / вопрос | Что проверить |
| --- | --- |
| Без прокси сайты не открываются | Это ожидаемая работа защиты |
| С прокси тоже не открываются | Слушатель 7897, выбранный VPN, доступность DNSTT-клиента/SSH SOCKS при выборе резерва |
| Сайты открываются, но IP провайдера | DIRECT в выбранной группе, режим Direct/Global, порядок правил, распознавание процесса |
| После обновления браузер выходит напрямую | Путь к `vivaldi.exe`, включённость правила и Firewall |
| Не открывается роутер `192.168.31.1` | Локальная сеть тоже заблокирована; использовать другой браузер |
| Есть `Allow` и `Block` для программы | Совпавшие блокирующие правила обычно приоритетнее; не создавать полный Block с отдельным Allow для loopback вместо этой схемы |
| Работает сторонний firewall | Проверить, действует ли Windows Firewall и его локальная политика; эта памятка не описывает сторонний продукт |

- Защита распространяется на указанный `vivaldi.exe`. Другие браузеры, отдельные загрузчики, обновлятор и внешние приложения требуют собственных правил.
- Системные DNS-запросы могут выполняться службой Windows от другого процесса. Это правило не гарантирует отсутствие DNS-утечек; DNS и Secure DNS проверяются отдельно.
- Firewall браузера не запрещает самому Clash использовать DIRECT. Поэтому YAML и проверка маршрута обязательны.
- Остановка VPN не должна автоматически переключать группу на DIRECT. Резервный узел допустим, если его выход тоже подходит.
- Успешный тест на ipify подтверждает конкретный запрос. Он не гарантирует анонимность и не исключает ограничений аккаунта по другим признакам.
- В случае смены System Proxy на TUN схему нужно перепроверить: поведение приложения и перехват трафика меняются.
- На втором ноутбуке определить его собственный путь Vivaldi и порт Clash, затем повторить установку и проверки.

## Источники

- [Mihomo — правила маршрутизации](https://wiki.metacubex.one/en/config/rules/)
- [Mihomo — общие настройки и определение процесса](https://wiki.metacubex.one/en/config/general/)
- [Mihomo — группа Select](https://wiki.metacubex.one/en/config/proxy-groups/select/)
- [Microsoft — New-NetFirewallRule](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule)
- [Vivaldi — Privacy and Security / WebRTC](https://help.vivaldi.com/desktop/privacy/privacy-settings/)

Дата фиксации рабочего сценария: **8 ноября 2026 года**.
