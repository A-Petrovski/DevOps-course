# Практические задания (Чекпоинты): Systemd, мониторинг нагрузки, Cron и управление процессами

## 1–2 и 4. Systemd unit для мониторинга `uptime`, `Load Average` и автоматической очистки логов

### Создание исполняемого скрипта (`sqript.sh`):
```bash
nano /home/amp/sqript.sh
```
```bash
#!/bin/bash

LOG_FILE="/home/amp/uptime_1_5_1.txt"
OVERLOAD_FILE="/home/amp/overload"
CLEANUP_LOG="/home/amp/cleanup"
MAX_SIZE=51200

while true; do
    UPTIME_OUT=\$(uptime)
    echo "\(UPTIME_OUT" >> "\)LOG_FILE"
    LOAD_1MIN=\$(awk '{print \$1}' /proc/loadavg)
    IS_OVERLOAD=\$(awk -v load_val="\$LOAD_1MIN" 'BEGIN {print (load_val > 1.0) ? 1 : 0}')
    if [ "\$IS_OVERLOAD" -eq 1 ]; then
        echo "\(UPTIME_OUT" >> "\)OVERLOAD_FILE"
        CURRENT_SIZE=\((stat -c\%s "\)OVERLOAD_FILE")
        if [ "\(CURRENT_SIZE" -ge "\)MAX_SIZE" ]; then
            if truncate -s 0 "\$OVERLOAD_FILE"; then
                echo "[\((date '+\%Y-\%m-\%d \%H:\%M:\%S')] file\)OVERLOAD_FILE cleaned" >> "\$CLEANUP_LOG"
            fi
        fi
    fi
    sleep 15
done
```
```bash
sudo chmod +x /home/amp/sqript.sh
```
- `chmod +x` — выдача прав на выполнение файла (`executable`), без которых `systemd` завершит службу с ошибкой `status=203/EXEC`.
- `>>` — дописывание вывода в конец файла без перезаписи предыдущих строк.
- `-v load_val="$LOAD_1MIN"` (в `awk`) — передача переменной из Bash в `awk` (используется `load_val`, так как слово `load` является зарезервированным в GNU Awk).
- `-eq` / `-ge` — операторы сравнения целых чисел в Bash («равно» и «больше или равно»).
- `stat -c%s` — получение точного размера файла в байтах без лишнего форматирования (`51200` байт = 50 КБ).
- `truncate -s 0` — обнуление размера файла до `0` байт без его удаления и с сохранением прав доступа.
- **Пример использования:** `sudo bash -x /home/amp/sqript.sh` — пошаговый отладочный запуск скрипта в терминале для поиска синтаксических ошибок.

### Создание конфигурации службы (`sqript.service`):
```bash
sudo nano /etc/systemd/system/sqript.service
```
```ini
[Unit]
Description=Uptime and Load Average Monitor
After=network.target

[Service]
Type=simple
ExecStart=/home/amp/sqript.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
- `After=network.target` — запуск службы только после инициализации сетевой подсистемы.
- `Type=simple` — основной процесс службы запускается напрямую командой из `ExecStart`.
- `Restart=always` — безусловный автоматический перезапуск процесса при любом завершении или принудительном убийстве (`kill`).
- `RestartSec=5` — пауза в 5 секунд перед перезапуском для защиты процессора от нагрузки при частых сбоях.
- `WantedBy=multi-user.target` — привязка службы к стандартному многопользовательскому режиму загрузки ОС.

### Активация службы и проверка перезапуска при убийстве процесса:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sqript.service
sudo systemctl status sqript.service
sudo systemctl kill -s SIGKILL sqript.service
sudo systemctl status sqript.service
```
- `daemon-reload` — перечитывание конфигурации `systemd` после создания или изменения `.service` файлов.
- `enable --now` — одновременное включение автозапуска при старте ОС и немедленный запуск службы.
- `kill -s SIGKILL` — отправка процессу сигнала принудительного завершения (`-9`) для проверки автоматического перезапуска и смены `Main PID`.
- **Пример использования:** `sudo journalctl -u sqript.service -f` — просмотр журнала работы конкретной службы в реальном времени.

---

## 3. Установка утилиты `stress` и нагрузочное тестирование

### Установка и запуск нагрузки на все ядра процессора:
```bash
sudo apt update && sudo apt install -y stress
nproc
stress --cpu \$(nproc) --timeout 50s
cat /home/amp/overload
```
- `-y` — автоматическое подтверждение установки пакета без запроса.
- `nproc` — вывод количества доступных процессорных ядер в системе (`x`).
- `--cpu $(nproc)` — автоматическая подстановка числа ядер для создания максимальной нагрузки.
- `--timeout 50s` — автоматическое завершение нагрузочного теста ровно через 50 секунд.
- **Пример использования:** `dd if=/dev/zero bs=1K count=51 | tr '\0' 'A' > /home/amp/overload` — искусственное наполнение файла до 51 КБ для быстрой проверки срабатывания очистки (пункт 4).

---

## 5. Настройка периодической проверки статуса юнита (`cron`)

### Добавление задачи в системный планировщик:
```bash
sudo crontab -e
```
```cron
*/10 * * * * echo "[\((date '+\%Y-\%m-\%d \%H:\%M:\%S')] sqript.service status:\)(systemctl is-active sqript.service)" >> /home/amp/cron_status.log
```
```bash
sudo crontab -l
```
- `-e` (в `crontab`) — открытие таблицы задач текущего пользователя (с `sudo` — пользователя `root`) для редактирования.
- `-l` (в `crontab`) — вывод списка всех активных расписаний без входа в редактор.
- `*/10 * * * *` — расписание запуска (каждые 10 минут, каждый час, каждый день).
- `is-active` (в `systemctl`) — вывод краткого текущего состояния службы (`active`, `inactive` или `failed`).
- `\%` — обязательное экранирование символа процента в `crontab` (без слеша `cron` воспринимает `%` как перенос строки).
- **Пример использования:** `*/10 * * * * systemctl is-active --quiet sqript.service || systemctl start sqript.service` — тихая проверка статуса с автоматическим запуском службы, если она упала.

---

## 6. Управление фоновыми и активными процессами (`ping`, `jobs`, `bg`, `fg`, `kill`)

### Перевод процесса между режимами, остановка и удаление:
```bash
ping 8.8.8.8
# Нажать Ctrl + Z (остановка процесса — статус Stopped)
bg
jobs
fg %1
# Нажать Ctrl + Z (остановка процесса перед удалением)
kill -9 %1
jobs
```
- `Ctrl + Z` — отправка активному процессу сигнала приостановки (`SIGTSTP`), переводящего его в состояние `Stopped` с возвратом терминала.
- `bg` — возобновление работы приостановленного процесса в фоновом режиме (`Background`).
- `jobs` — отображение списка фоновых и остановленных задач текущей сессии с их порядковыми номерами (`[1]`, `[2]`).
- `fg %1` — возврат задачи с номером `1` из фонового режима на передний план (`Foreground`).
- `kill -9 %1` — принудительное удаление процесса по его номеру задачи в `jobs` с отправкой сигнала `SIGKILL` (`-9`).
- **Пример использования:** `ping 8.8.8.8 > /dev/null &` — запуск утилиты сразу в фоновом режиме с перенаправлением вывода в `/dev/null`, чтобы строки не засоряли экран терминала.

---

## 7. Сохранение истории использованных команд в отдельный файл

### Выгрузка истории команд текущей сессии:
```bash
history -a
history | tail -n 50 > /home/amp/commands_1_5.txt
```
- `-a` (в `history`) — принудительная запись команд из оперативной памяти текущей сессии в файл `~/.bash_history`.
- `tail -n 50` — выборка последних 50 выполненных команд.
- `>` — сохранение результата в новый текстовый файл отчёта.
- **Пример использования:** `history | grep -E "systemctl|stress|crontab" > filtered_commands.txt` — сохранение в файл только тех команд, которые относятся к конкретным утилитам.
