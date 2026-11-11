# Практические задания (Чекпоинты)

## 1. Создание пользователей
```bash
sudo adduser raymond
sudo adduser john
```
- `adduser` — интерактивное создание пользователя с домашней директорией.

## 2. Настройка SSH и передача ключа (SCP)

### На хосте (Windows PowerShell):
```powershell
ssh-keygen -t ed25519 -C "raymond@vm" -f "$env:USERPROFILE\.ssh\raymond_key"
scp -O "$env:USERPROFILE\.ssh\raymond_key.pub" amp@192.168.1.18:/tmp/raymond_key.pub
```
- `-t ed25519` — выбор алгоритма шифрования.
- `-C` — комментарий в конце публичного ключа.
- `-f` — путь и имя файла создаваемого ключа.
- `-O` — использование классического протокола SCP вместо SFTP.

### На сервере (Ubuntu):
```bash
sudo passwd -d raymond
sudo mkdir -p /home/raymond/.ssh
sudo mv /tmp/raymond_key.pub /home/raymond/.ssh/authorized_keys
sudo chown -R raymond:raymond /home/raymond
sudo chmod 750 /home/raymond
sudo chmod 700 /home/raymond/.ssh
sudo chmod 600 /home/raymond/.ssh/authorized_keys
```
- `-d` — удаление пароля и снятие системной блокировки аккаунта.
- `-p` — создание директории вместе с родительскими папками без ошибки, если она уже есть.
- `-R` — рекурсивное применение к папке и всему содержимому.

### Конфигурация `/etc/ssh/sshd_config`:
```text
Match User raymond
    PasswordAuthentication no
    PubkeyAuthentication yes

Match User john
    PasswordAuthentication yes
    PubkeyAuthentication no
```
```bash
sudo sshd -t && sudo systemctl restart ssh
```
- `-t` — проверка синтаксиса конфигурации перед перезапуском.

## 3. Настройка прав `sudo`
```bash
sudo usermod -aG sudo raymond
echo "raymond ALL=(ALL:ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/raymond
sudo chmod 440 /etc/sudoers.d/raymond
sudo visudo -c
```
- `-aG` — добавление пользователя в дополнительную группу без удаления из текущих.
- `-c` — проверка синтаксиса файлов `sudoers`.

## 4. Права доступа к файлам и shell-скриптам (ACL)

### Документ `Raymond` (чтение для `John`, без записи):
```bash
echo "Secret data" | sudo -u raymond tee /home/raymond/raymonds_file.txt
sudo apt update && sudo apt install -y acl
sudo setfacl -m u:john:x /home/raymond
sudo setfacl -m u:john:r /home/raymond/raymonds_file.txt
```
- `-u raymond` — выполнение команды от имени указанного пользователя.
- `-y` — автоматическое подтверждение установки пакета.
- `-m` — изменение/добавление правила ACL (`x` — проход через папку, `r` — чтение файла).

### Скрипт `John` (запуск для `Raymond`):
```bash
sudo -u john bash -c 'printf "#!/bin/bash\necho \"Hello!\"\n" > /home/john/hello.sh'
sudo setfacl -m u:raymond:x /home/john
sudo setfacl -m u:raymond:rx /home/john/hello.sh
sudo -u raymond /home/john/hello.sh
```
- `-c` — передача строки команд для выполнения оболочке.
- `rx` — права на чтение и выполнение (оба обязательны для запуска скрипта).

## 5. Смена интерпретатора по умолчанию
```bash
sudo usermod -s /bin/bash john
sudo useradd -m -s /bin/sh george
sudo passwd george
awk -F: '\$3 >= 1000 && \$1 != "nobody" {print \$1 " -> " \$7}' /etc/passwd
```
- `-s` — задание пути к интерпретатору по умолчанию.
- `-m` — создание домашней директории при использовании `useradd`.
- `-F:` — указание двоеточия как разделителя колонок в `awk`.

## 6. Группы и запуск скрипта новым пользователем
```bash
sudo usermod -aG john george
sudo chmod 750 /home/john
sudo setfacl -m g::rx /home/john/hello.sh
sudo -u george /home/john/hello.sh
```
- `g::rx` — выдача базовой группе файла прав на чтение и выполнение.

## 7. Передача файла с хоста через SCP
```powershell
scp -O "F:\Soft\shared\shared_by_scp.txt" amp@192.168.1.18:/home/amp/
```
```bash
sudo mv /home/amp/shared_by_scp.txt /home/
```
