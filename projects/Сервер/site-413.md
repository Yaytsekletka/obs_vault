RUS:
Панель управления 3X-UI (https://github.com/MHSanaei/3x-ui) доступна по следующим данным:
Ссылка - http://95.181.167.188:61310
Логин - paneladmin
Пароль - aEOgv613D5__
```
ssh root@95.181.167.188
```
```
GIT_SSH_COMMAND="ssh -i ~/.ssh/site-deploy-key" git clone git@github.com-site:site-413-git/base.git
```
```
GIT_SSH_COMMAND="ssh -i ~/.ssh/site-deploy-key" git pull
```
Узнать порты которые слушает сервер
```
sudo ss -tuln
```

![[Pasted image 20250412200531.png]]
### 🧠 **Сеть и соединения**
#### 1. `sudo ss -tuln`
Уже знаешь — показывает **открытые порты и сокеты** (TCP/UDP).
#### 2. `sudo netstat -tulnp`
Альтернатива `ss`, выводит то же, но с **PID/имя процесса** (можно установить через `sudo apt install net-tools`).
#### 3. `sudo lsof -i -n -P`
Показывает **кто слушает порты**, например:
```bash
sudo lsof -iTCP -sTCP:LISTEN
```
#### 4. `sudo tcpdump -i any`
Мощная штука для **прослушки трафика**, например:
```bash
sudo tcpdump port 80
```
### 💻 **Процессы и ресурсы**
#### 5. `htop`
Красивый **интерактивный монитор** процессов (установи через `sudo apt install htop`).  
Можно видеть:
- сколько CPU и RAM жрёт nginx или бот
- сортировка по нагрузке
- удобно убивать процессы
#### 6. `top`
Более простая встроенная альтернатива `htop`.
#### 7. `ps aux --sort=-%mem | head`
Показывает **топ процессов по использованию памяти**.
#### 8. `ps aux --sort=-%cpu | head`
Показывает **топ по загрузке CPU**.
### 📁 **Диск и файлы**
#### 9. `df -h`
Показывает **занятое место на диске**, с человеко-читаемыми единицами (`-h`).
#### 10. `du -sh /var/www/*`
Показывает **размер каждой папки**, чтобы найти кто жрёт место.
### 🛡️ **Безопасность и пользователи**

#### 11. `last`
Кто и когда логинился по SSH.
#### 12. `who`
Кто **сейчас в системе**.
#### 13. `sudo journalctl -xe`
Показывает **системные ошибки и важные события** (особенно полезно после падения чего-то).
### 🌐 **Проверка подключения**

#### 14. `ping ya.ru`

Проверка связи с интернетом.

#### 15. `curl -I http://localhost`

Проверяет, отвечает ли твой сайт локально (возвращает заголовки).

#### 16. `traceroute ya.ru`

Путь до сайта — полезно, если где-то в сети затык.







### **Скрипт проверяющий удалена ли папка с гитом**
скрипт который проверяет не удалилась ли папка с проектом и если удалилась - присылает сообщение в телеграмме на указанный id
/usr/local/bin/watch_website.sh
```
FOLDER="/var/www/base"
BOT_TOKEN="___"
CHAT_ID="869267731"

inotifywait -m -e delete_self,move_self "$FOLDER" |
while read path action file; do
  curl -s -X POST "https://api.telegram.org/bot$BOT_TOKEN/sendMessage" \
    -d chat_id="$CHAT_ID" \
    -d text="⚠️ Папка $FOLDER была удалена или перемещена! Действие: $action"
done
```
Работоспособность этого скрипта неизвестна; работает через nohup


### **Бот для git pull-а и системных команд**
 systemd
`/etc/systemd/system/system-bot.service`
```
[Unit]
Description=Telegram System Bot

[Service]
WorkingDirectory=/var/www/base
ExecStart=/usr/bin/python3 /var/www/base/system_bot.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target

```
`Created symlink /etc/systemd/system/multi-user.target.wants/system-bot.service → /etc/systemd/system/system-bot.service.`

Запустить бота `sudo systemctl start system-bot.service`
запускается автоматически при перезагрузке сервера благодаря systemd

📡 Использование
- Чтобы подтянуть изменения:  
    В Telegram напишите:
    ```
    /update feature/new-page
    ```
    ```
    /update
    ```
    если хотите пуллить изменения из main.
- Чтобы узнать запущенные процессы:  
    Напишите:
    ```
    /procs
    ```
- Чтобы узнать список открытых портов:  
    Напишите:
    ```
    /ports
    ```
- Чтобы проверить использование диска:  
    Напишите:
    ```
    /disk
    ```


