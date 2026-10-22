# VPS NL — Marzban / Xray

> [!info] Сервер  
> **Провайдер:** serv.host  
> **Тариф:** Ryzen-9-5950X-0.5  
> **Дата открытия:** 29.09.2026  
> **Хост:** `arkornrf1.serv.host`  
> **IPv4:** `13.143.193.76`  
> **ОС:** Ubuntu 24.04  
> **SSH-пользователь:** `root`  
> **SSH-порт:** `22`  
> **Панель:** Marzban  
> **VPN:** Xray VLESS + REALITY + Vision  
> **Порт панели:** `8000`  
> **Порт VPN:** `443`

---

## 1. Подключение — PowerShell

**Открыть SSH-сессию:**

```
ssh root@13.143.193.76
```

**Открыть туннель к панели:**

```
ssh -N -L 127.0.0.1:8000:127.0.0.1:8000 root@13.143.193.76
```

Панель: [http://localhost:8000/dashboard/](http://localhost:8000/dashboard/)

`-N` — не открывать командную оболочку, только туннель.  
`-L` — пробросить локальный порт на VPS.  
`Ctrl+C` — закрыть туннель.

**Проверить доступность SSH:**

```
Test-NetConnection 13.143.193.76 -Port 22
```

**Проверить порт VPN:**

```
Test-NetConnection 13.143.193.76 -Port 443
```

**Проверить DNS домена:**

```
Resolve-DnsName arkornrf1.serv.host
```

**Диагностика SSH с подробным выводом:**

```
ssh -vvv root@13.143.193.76
```

**Копировать файл с VPS на компьютер:**

```
scp root@13.143.193.76:/path/file.txt "$HOME\Downloads\"
```

---

## 2. Управление Marzban — SSH

**Проверить состояние:**

```
marzban status
```

**Просмотреть логи:**

```
marzban logs
```

**Запустить:**

```
marzban up
```

**Перезапустить:**

```
marzban restart
```

**Остановить — отключит VPN:**

```
marzban down
```

**Обновить Marzban — предварительно создать бэкап:**

```
marzban update
```

**Создать резервную копию:**

```
marzban backup
```

**Узнать остальные команды:**

```
marzban help
```

---

## 3. Docker / Xray

**Список запущенных контейнеров:**

```
docker ps
```

**Все контейнеры, включая остановленные:**

```
docker ps -a
```

**Использование ресурсов контейнерами:**

```
docker stats --no-stream
```

**Последние 100 строк логов:**

```
docker logs --tail 100 marzban
```

**Непрерывный просмотр логов:**

```
docker logs -f --tail 50 marzban
```

**Узнать версию Docker:**

```
docker --version
```

**Узнать текущую версию Xray (если бинарник расположен по стандартному пути Marzban):**

```
docker exec marzban /usr/local/bin/xray version
```

> [!note]  
> Название контейнера и путь до Xray необходимо проверить через `docker ps` и `docker exec`, поскольку они зависят от установки.

---

## 4. Состояние VPS

|Команда|Назначение|
|---|---|
|`uptime`|Время работы и средняя нагрузка|
|`free -h`|Использование оперативной памяти|
|`df -h`|Свободное место на дисках|
|`df -i`|Свободные inode|
|`htop`|Интерактивный мониторинг процессов|
|`top`|Мониторинг процессов без дополнительных пакетов|
|`lsblk`|Диски и разделы|
|`uname -r`|Версия ядра Linux|
|`cat /etc/os-release`|Версия ОС|
|`ip -br addr`|Сетевые интерфейсы и адреса|
|`ip route`|Маршрутизация|
|`timedatectl`|Время и часовой пояс|

**Топ процессов по потреблению ОЗУ:**

```
ps aux --sort=-%mem | head -15
```

**Топ процессов по нагрузке CPU:**

```
ps aux --sort=-%cpu | head -15
```

**Размер каталогов Marzban:**

```
du -sh /var/lib/marzban /opt/marzban 2>/dev/null
```

---

## 5. Сеть и диагностика VPN

**Какие порты прослушивает сервер:**

```
ss -tulpen
```

**Проверить порт 443:**

```
ss -tlnp | grep ':443'
```

**Проверить порт панели 8000:**

```
ss -tlnp | grep ':8000'
```

**Проверить SSH-службу:**

```
systemctl status ssh
```

**Проверить DNS-резолвинг:**

```
getent hosts github.com
```

**Проверить исходящий HTTPS:**

```
curl -I --max-time 10 https://github.com
```

**Посмотреть правила UFW:**

```
ufw status verbose
```

**Посмотреть системный журнал за последние 30 минут:**

```
journalctl --since "30 minutes ago" -p warning --no-pager
```

**Проверить неудачные авторизации по SSH:**

```
journalctl -u ssh --since today --no-pager
```

---

## 6. Бэкапы и восстановление

**Создать резервную копию Marzban:**

```
marzban backup
```

**Проверить содержимое каталога данных:**

```
ls -lah /var/lib/marzban
```

**Найти файлы резервных копий:**

```
find /var/lib/marzban /opt/marzban -type f \( -iname '*.zip' -o -iname '*.tar.gz' -o -iname '*.bak' \) 2>/dev/null
```

**Создать локальный архив конфигурации и данных (содержит секреты):**

```
tar -czf /root/marzban-manual-backup.tar.gz /opt/marzban /var/lib/marzban
```

> [!warning] Важно  
> Резервная копия должна существовать **за пределами VPS**. Локальный архив не спасёт от блокировки доступа или удаления сервера. При активных изменениях SQLite-файлов простой файловый архив не гарантирует согласованную копию базы.

**Дополнительно зафиксировать:**

- Путь хранения штатных бэкапов.
    
- Настройки Telegram-бота для бэкапов.
    
- Периодичность выгрузки.
    
- Где хранится внешняя копия.
    
- Порядок восстановления на новом VPS.
    

---

## 7. Управление хостом

**Проверить обновления пакетов:**

```
apt update
apt list --upgradable
```

**Установить обновления:**

```
apt upgrade
```

**Проверить необходимость перезагрузки:**

```
test -f /var/run/reboot-required && echo "Reboot required" || echo "No reboot requested"
```

**Перезагрузить VPS — временно отключит все сервисы:**

```
reboot
```

**Выключить VPS — для повторного запуска может потребоваться кабинет провайдера:**

```
shutdown -h now
```

---

## 8. Если VPN перестал работать

1. Проверить SSH-доступ из PowerShell: `Test-NetConnection`.
    
2. Подключиться по SSH.
    
3. Проверить `uptime`, `free -h`, `df -h`.
    
4. Проверить `marzban status` и `docker ps`.
    
5. Посмотреть `marzban logs`.
    
6. Проверить, слушается ли порт 443.
    
7. При необходимости создать бэкап и выполнить `marzban restart`.
    
8. Повторно проверить VPN через Wi-Fi и мобильную сеть.
    
9. Если SSH недоступен — проверить веб-консоль VPS у провайдера.
    

> [!danger] Не выполнять без понимания последствий  
> `ufw enable`, изменение SSH-порта, `iptables -F`, `docker system prune -a`, `docker compose down -v`, удаление каталогов Marzban и смена конфигурации Xray. Эти операции могут оборвать доступ или удалить данные.

---

## 9. Карта инфраструктуры — заполнить

| Компонент              | Значение                                                                  |
| ---------------------- | ------------------------------------------------------------------------- |
| Marzban ENV            | `/opt/marzban/.env`                                                       |
| Docker Compose         | `/opt/marzban/docker-compose.yml`                                         |
| Данные Marzban         | `/var/lib/marzban/`                                                       |
| Xray config            | `/var/lib/marzban/xray_config.json`                                       |
| Xray version           | 24.12.31 (Xray, Penetrates Everything.)<br>4be32e9 (go1.23.4 linux/amd64) |
| REALITY SNI            | `github.com`                                                              |
| Telegram backup        | Ежедневно в 00:00 UTC                                                     |
| Дополнительные сервисы | -                                                                         |
| Web-консоль провайдера | serv.host                                                                 |
| Хранение паролей       | -                                                                         |

---

## 10. SSH-алиас — чтобы не вводить IP каждый раз

На Windows открой файл `$HOME\.ssh\config` (создай, если отсутствует) и добавь:

```
Host arkorn-nl
    HostName 13.143.193.76
    User root
    Port 22
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

После этого в PowerShell достаточно:

```
ssh arkorn-nl
```

А для панели:

```
ssh -N -L 127.0.0.1:8000:127.0.0.1:8000 arkorn-nl
```

## 11. Разовая проверка всего сервера

*Одна команда из PowerShell выдаст основные показатели без отдельного интерактивного входа:*

```
ssh arkorn-nl "uptime; free -h; df -h /; docker ps; marzban status"
```

**Особенно полезно после перебоев в работе VPN.*