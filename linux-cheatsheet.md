---
title: "Linux Production Cheatsheet: Troubleshooting & Administration"
version: "2.1.0"
last_updated: "2025-01-01"
author: "Senior Linux/DevOps Engineer"
---

# Linux Production Cheatsheet

> **Цель:** Практическое руководство для администрирования production-систем, диагностики инцидентов и устранения неисправностей. Команды проверены на реальных инцидентах.
>
> 🔐 = требует root / sudo | 💣 = деструктивная операция | ⚡ = срочная диагностика

---

## Содержание

1. [Навигация и файловая система](#1-навигация-и-файловая-система)
2. [Работа с текстом и потоками](#2-работа-с-текстом-и-потоками)
3. [Управление процессами](#3-управление-процессами)
4. [Права доступа и безопасность](#4-права-доступа-и-безопасность)
5. [Архивирование и передача файлов](#5-архивирование-и-передача-файлов)
6. [Планировщики задач](#6-планировщики-задач)
7. [Управление пакетами](#7-управление-пакетами)
8. [Сетевые инструменты](#8-сетевые-инструменты)
9. [Firewall и безопасность сети](#9-firewall-и-безопасность-сети)
10. [SSH и удалённый доступ](#10-ssh-и-удалённый-доступ)
11. [Systemd и управление сервисами](#11-systemd-и-управление-сервисами)
12. [Мониторинг ресурсов](#12-мониторинг-ресурсов)
13. [Дисковая подсистема](#13-дисковая-подсистема)
14. [Ядро и система](#14-ядро-и-система)
15. [🔥 TROUBLESHOOTING](#15-troubleshooting)
16. [Быстрый справочник](#быстрый-справочник)
17. [Дополнительные материалы](#дополнительные-материалы)

---

## 1. Навигация и файловая система

Эффективная работа с ФС — основа скорости диагностики в production. Умение быстро находить файлы по содержимому, правам и времени изменения сокращает время инцидента в разы. Знание inode и extended attributes помогает расследовать аномалии безопасности.

```bash
# Поиск по имени и типу
find /var/log -name "*.log" -type f -mtime -1          # логи за последние 24 часа
find /etc -name "*.conf" -newer /etc/passwd            # конфиги новее passwd
find / -perm -4000 -type f 2>/dev/null                 # SUID файлы (аудит безопасности)
find / -perm -2000 -type f 2>/dev/null                 # SGID файлы
find /tmp -user root -perm /o+w 2>/dev/null            # writable root-файлы в /tmp

# Поиск по размеру
find /var -size +100M -type f                          # файлы > 100MB
find / -size +1G -type f 2>/dev/null                   # файлы > 1GB

# Поиск по содержимому
grep -r "OOMKilled" /var/log/ --include="*.log" -l     # файлы с OOM событиями
grep -rn "password" /etc/ --include="*.conf" 2>/dev/null  # пароли в конфигах

# Дисковое пространство
df -hT                                                 # диски с типом ФС
df -i                                                  # использование inode
du -sh /var/log/* | sort -rh | head -20                # топ директорий по размеру
du -sh /* 2>/dev/null | sort -rh | head -10            # топ на корневой ФС

# Информация об inode
ls -li /etc/passwd                                     # inode номер файла
stat /var/log/syslog                                   # полная информация о файле
lsattr /etc/passwd                                     # extended attributes

# Работа с симлинками
find /etc -type l -ls                                  # все симлинки в /etc
readlink -f /etc/alternatives/java                     # полный путь симлинка
ls -la /proc/self/exe                                  # на что указывает /proc/self/exe

# Extended attributes
getfattr -d /etc/passwd                                # показать xattr
setfattr -n user.comment -v "production" /etc/myapp   # установить xattr
```

```bash
# locate (требует обновления базы)
🔐 updatedb                                            # обновить базу locate
locate -i "*.conf" | grep nginx                        # быстрый поиск
locate --statistics                                    # статистика базы

# Навигация и просмотр структуры
tree -L 2 /etc/nginx                                   # дерево каталогов, 2 уровня
tree -sh /var/log                                      # с размерами файлов
ls -lath /var/log/ | head -20                          # новые файлы первыми
```

> ⚠️ **Антипаттерн:** Использование `find / -name filename` без ограничения области поиска (-path исключений) на production — зависает на NFS-маунтах и огромных директориях. Всегда добавляй `-xdev` для ограничения одной ФС или явно исключай `/proc`, `/sys`: `find / -not \( -path /proc -prune \) -not \( -path /sys -prune \)`.

> ✅ **Senior-совет:** Когда диск полон, а `du` не показывает виновника — ищи файлы, которые удалены, но открыты процессами: `lsof +L1 | grep deleted` или `lsof | grep "(deleted)"`. Такие файлы занимают место, но невидимы для `ls`. Перезапуск процесса освобождает место немедленно.

---

## 2. Работа с текстом и потоками

Pipeline-обработка текста — ключевой навык для анализа логов в реальном времени без загрузки в память. Комбинация grep/awk/sed позволяет обрабатывать файлы в сотни гигабайт на лету.

```bash
# grep — поиск с контекстом
grep -n "ERROR" /var/log/app.log                       # с номерами строк
grep -c "CRITICAL" /var/log/app.log                    # подсчёт совпадений
grep -v "DEBUG" /var/log/app.log                       # исключить DEBUG
grep -A 5 -B 2 "FATAL" /var/log/app.log                # 5 строк после, 2 до
grep -E "ERROR|WARN|CRITICAL" /var/log/app.log         # расширенные regex
grep -P "\d{4}-\d{2}-\d{2} \d{2}:5[0-9]:" app.log     # Perl regex (время 50-59 мин)
grep "timeout" /var/log/nginx/error.log | grep -oP 'upstream: \K[^ ]+' | sort | uniq -c

# Обработка больших файлов — без загрузки в память
tail -n 1000 /var/log/app.log | grep ERROR             # последние 1000 строк
zcat /var/log/syslog.1.gz | grep "kernel"              # сжатый лог
zgrep "OOM" /var/log/syslog*.gz                        # поиск в архивах

# awk — обработка колонок и агрегация
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head # топ IP
awk '$9 == 500 {print $0}' /var/log/nginx/access.log   # только 500 ошибки
awk '{sum+=$NF} END {print "Total:", sum}' latency.log # суммирование последней колонки
awk -F: '$3 >= 1000 {print $1, $3}' /etc/passwd        # обычные пользователи (UID >= 1000)
awk 'NR%1000==0 {print NR, $0}' bigfile.log            # каждая 1000-я строка

# Мониторинг логов в реальном времени с фильтрацией
tail -f /var/log/app.log | grep --line-buffered "ERROR\|WARN"
tail -f /var/log/nginx/access.log | awk --line-buffered '{if ($9>=500) print $0}'

# sed — потоковое редактирование
sed -n '100,200p' /var/log/app.log                     # строки 100-200
sed '/^#/d; /^$/d' /etc/nginx/nginx.conf               # убрать комментарии и пустые строки
sed 's/password=.*/password=REDACTED/g' config.txt     # маскировать пароли
sed -i.bak 's/old-server/new-server/g' /etc/app/*.conf # заменить с backup

# cut, sort, uniq
cut -d: -f1,7 /etc/passwd                             # имя и оболочка пользователей
sort -t: -k3 -n /etc/passwd                           # сортировка по UID
sort -rn latency.log | head -20                        # топ задержек
uniq -d sorted.log                                     # только дублирующиеся строки
uniq -c sorted.log | sort -rn | head -10               # частота строк

# tee — запись и передача дальше
command 2>&1 | tee /var/log/deploy-$(date +%Y%m%d).log # лог + консоль одновременно
tail -f app.log | tee >(grep ERROR > errors.log) | grep WARN > warns.log

# xargs — передача аргументов
find /tmp -name "*.tmp" -mtime +7 | xargs rm -f        # удалить старые tmp
find /etc -name "*.conf" | xargs grep -l "192.168.1.1" # конфиги с IP
cat ips.txt | xargs -P 4 -I{} ping -c 1 {}             # параллельный ping

# tr — замена символов
tr '[:lower:]' '[:upper:]' < input.txt                 # в верхний регистр
tr -d '\r' < windows.txt > unix.txt                    # убрать Windows CRLF
cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32    # случайная строка 32 символа
```

```bash
# Pipeline для анализа access.log (production example)
awk '{print $7}' /var/log/nginx/access.log \
  | sort | uniq -c | sort -rn | head -20               # топ URL

# Подсчёт ошибок по часам
grep "error" /var/log/app.log \
  | awk '{print $1, substr($2,1,2)}' \
  | sort | uniq -c                                     # ошибки по дате и часу

# Быстрый анализ времени ответа из nginx
awk '$NF > 1.0' /var/log/nginx/access.log \
  | awk '{print $7, $NF}' \
  | sort -k2 -rn | head -20                            # медленные запросы > 1 сек
```

> ⚠️ **Антипаттерн:** `cat bigfile.log | grep pattern` — лишний процесс. Используй `grep pattern bigfile.log`. Для очень больших файлов: `grep pattern <(zcat bigfile.log.gz)` или chunk-обработку через `split`.

> ✅ **Senior-совет:** При анализе логов в production всегда добавляй `--line-buffered` к grep и `flush()` к awk при работе с `tail -f`. Без этого буферизация stdout ломает pipeline и ты видишь данные с задержкой или вообще ничего — особенно критично при диагностике инцидента в реальном времени.

---

## 3. Управление процессами

Умение быстро идентифицировать и управлять процессами — критичный навык при инцидентах. Понимание сигналов, cgroups и limits предотвращает повторные инциденты.

```bash
# Просмотр процессов
ps aux --sort=-%cpu | head -20                         # топ по CPU
ps aux --sort=-%mem | head -20                         # топ по памяти
ps -eo pid,ppid,user,%cpu,%mem,vsz,rss,stat,start,time,comm --sort=-%cpu | head
ps -ef --forest                                        # дерево процессов
ps -p 1234 -o pid,ppid,user,command                    # информация о конкретном PID

# pgrep / pkill
pgrep -la nginx                                        # PID и командная строка nginx
pgrep -u www-data                                      # процессы пользователя
pkill -HUP nginx                                       # перечитать конфиг nginx
pkill -9 -u baduser 2>/dev/null                        # убить все процессы пользователя
pgrep -x "defunct" | wc -l                             # количество zombie-процессов

# Управление приоритетами
nice -n 19 ./backup.sh                                 # запуск с низким приоритетом
🔐 renice -n -5 -p 1234                                # повысить приоритет (только root)
renice -n 10 -u backupuser                             # снизить приоритет всем процессам

# Фоновые задачи
nohup long_running_script.sh > /var/log/script.log 2>&1 &  # устойчиво к разрыву SSH
disown -h %1                                           # отвязать от терминала
jobs -l                                                # список фоновых задач
bg %1                                                  # продолжить в фоне
fg %1                                                  # вернуть на передний план

# Сигналы
kill -l                                                # список сигналов
kill -HUP 1234    # 1  — перечитать конфиг
kill -TERM 1234   # 15 — вежливое завершение (по умолчанию)
kill -KILL 1234   # 9  — принудительное завершение (нет cleanup)
kill -STOP 1234   # 19 — заморозить процесс
kill -CONT 1234   # 18 — разморозить процесс
kill -USR1 1234   # пользовательский сигнал (logrotate для nginx)
kill -USR2 1234   # пользовательский сигнал (graceful reload для некоторых сервисов)

# cgroups v2 — лимиты
🔐 systemctl set-property myservice.service CPUQuota=50%     # лимит CPU 50%
🔐 systemctl set-property myservice.service MemoryMax=512M   # лимит памяти
cat /sys/fs/cgroup/system.slice/myservice.service/cpu.stat   # статистика CPU

# ulimit
ulimit -a                                              # все лимиты текущей оболочки
ulimit -n 65536                                        # файловые дескрипторы (текущая сессия)
cat /proc/1234/limits                                  # лимиты конкретного процесса
```

```bash
# htop — интерактивный мониторинг
# F6 — выбор поля сортировки
# F5 — дерево процессов
# u — фильтр по пользователю
# / — поиск по имени
# k — отправить сигнал

# Поиск zombie-процессов и их родителей
ps aux | awk '$8 ~ /^Z/ {print $2}' | while read pid; do
  ppid=$(ps -o ppid= -p $pid)
  echo "Zombie: $pid, Parent: $ppid ($(ps -o comm= -p $ppid))"
done
```

> ⚠️ **Антипаттерн:** `kill -9` как первый инструмент. `SIGKILL` не даёт процессу освободить ресурсы, закрыть соединения БД, сбросить буферы на диск. Всегда начинай с `SIGTERM`, жди 10-30 секунд, потом `SIGKILL` если не помогло.

> ✅ **Senior-совет:** Zombie-процессы не занимают CPU/RAM, но занимают PID. Если накопилось много zombie — значит родительский процесс не вызывает `wait()`. Убей родителя (если он не нужен) или перезапусти его — zombie исчезнут автоматически. Количество zombie само по себе — симптом проблемы в приложении, а не причина.

---

## 4. Права доступа и безопасность

Правильная настройка прав — основа безопасности системы. ACL и capabilities позволяют реализовать принцип минимальных привилегий без добавления пользователей в группы.

```bash
# chmod — базовые права
chmod 644 /etc/app/config.yml                          # rw-r--r--
chmod 750 /opt/app/bin/start.sh                        # rwxr-x---
chmod u+x,g-w,o-rwx script.sh                         # символьная форма
chmod -R 755 /var/www/html                             # рекурсивно
chmod +t /tmp                                          # sticky bit

# Специальные биты
chmod 4755 /usr/bin/myprogram                          # SUID
chmod 2755 /usr/bin/myprogram                          # SGID
find / -perm /4000 -type f 2>/dev/null                 # все SUID файлы

# chown
chown app:app /opt/app -R                              # рекурсивно
chown --reference=/etc/passwd /tmp/newfile             # права как у файла-образца

# umask
umask                                                  # текущая маска
umask 027                                              # файлы 640, директории 750
umask -S                                               # символьная форма

# ACL (Access Control Lists)
getfacl /var/log/app/                                  # текущие ACL
setfacl -m u:deploy:rx /var/log/app/                   # дать права deploy без группы
setfacl -m g:developers:rw /opt/project/               # права для группы
setfacl -d -m u:deploy:rx /var/log/app/                # default ACL для новых файлов
setfacl -R -m u:ci:rx /opt/project/                    # рекурсивно
setfacl -x u:deploy /var/log/app/                      # удалить ACL для пользователя
getfacl /source | setfacl --set-file=- /destination   # копировать ACL

# chattr — неизменяемые файлы
🔐 chattr +i /etc/passwd                               # immutable — нельзя изменить даже root
🔐 chattr -i /etc/passwd                               # снять immutable
🔐 chattr +a /var/log/app.log                          # append-only (только добавлять)
lsattr /etc/passwd                                     # просмотр атрибутов

# sudo
sudo -l                                                # что разрешено текущему пользователю
sudo -l -U username                                    # для другого пользователя
sudo -i                                                # интерактивная root-сессия
sudo -u www-data whoami                                # выполнить от другого пользователя
visudo                                                 # безопасное редактирование sudoers
🔐 sudo -k                                             # сбросить кэш пароля sudo

# capabilities (Linux capabilities)
getcap /usr/bin/ping                                   # capabilities файла
🔐 setcap cap_net_raw+ep /usr/bin/myapp                # дать raw socket без root
🔐 setcap -r /usr/bin/myapp                            # удалить capabilities
capsh --print                                          # capabilities текущего процесса
cat /proc/1234/status | grep -i cap                    # capabilities процесса

# SELinux
getenforce                                             # Enforcing/Permissive/Disabled
sestatus                                               # детальный статус
setenforce 0                                           # временно в Permissive
ls -Z /etc/nginx/nginx.conf                            # SELinux контекст файла
ps axZ | grep nginx                                    # контекст процессов
ausearch -m avc -ts recent | audit2why                 # анализ отказов SELinux
audit2allow -a -M mymodule                             # создать модуль разрешений

# AppArmor (Debian/Ubuntu)
aa-status                                              # статус AppArmor
aa-complain /usr/sbin/nginx                            # режим жалоб (логи без блокировки)
aa-enforce /usr/sbin/nginx                             # режим принуждения
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx      # перезагрузить профиль
```

> ⚠️ **Антипаттерн:** `chmod -R 777 /var/www` — классическая ошибка при отладке прав. Вместо этого выясни реальную проблему через `sudo -u www-data ls -la /path`, `namei -om /path/to/file`. Права 777 в production — прямой путь к компрометации.

> ✅ **Senior-совет:** `namei -om /var/www/app/upload` показывает цепочку прав по всему пути — это быстрее, чем проверять каждую директорию вручную. Спасает когда проблема с правами на промежуточной директории в длинном пути.

---

## 5. Архивирование и передача файлов

Архивирование с проверкой целостности — обязательное условие надёжного бэкапа. rsync с правильными флагами экономит трафик и время при инкрементальных передачах.

```bash
# tar — архивирование
tar -czf backup-$(date +%Y%m%d).tar.gz /etc /home      # создать gzip-архив
tar -cjf backup.tar.bz2 /opt/app                       # создать bzip2-архив (меньше, медленнее)
tar -cJf backup.tar.xz /opt/app                        # xz (максимальное сжатие)
tar -tvf archive.tar.gz | head -30                     # содержимое без распаковки
tar -xzf archive.tar.gz -C /tmp/restore/               # распаковать в директорию
tar -xzf archive.tar.gz --strip-components=1           # убрать верхний уровень
tar -czf - /opt/app | ssh user@backup-server "cat > /backup/app.tar.gz"  # по SSH на лету

# Проверка целостности
md5sum backup.tar.gz > backup.tar.gz.md5
sha256sum backup.tar.gz > backup.tar.gz.sha256
sha256sum -c backup.tar.gz.sha256                      # проверить

# rsync — инкрементальная синхронизация
rsync -avz --progress /opt/app/ user@backup:/backup/app/         # с прогрессом
rsync -avz --delete /var/www/ user@standby:/var/www/             # с удалением лишних
rsync -avz --exclude='*.log' --exclude='tmp/' /opt/app/ backup:  # с исключениями
rsync -avz --link-dest=/backup/app/2025-01-01 /opt/app/ /backup/app/$(date +%Y-%m-%d)/ # инкрементал с hard links
rsync -n -avz /src/ /dst/                              # dry run — показать что изменится
rsync --bwlimit=10000 -avz /large/ backup:/large/       # ограничить до 10 MB/s
rsync -avz -e "ssh -i ~/.ssh/backup_key -p 2222" /src/ user@host:/dst/

# scp
scp -r -P 2222 /opt/app user@backup:/backup/            # рекурсивно, нестандартный порт
scp -C user@remote:/tmp/dump.sql /tmp/                  # с сжатием

# Проверка занимаемого места до/после
du -sh /opt/app                                        # исходный размер
ls -lh backup.tar.gz                                   # размер архива
# Коэффициент сжатия: (original - compressed) / original * 100%

# Стратегия бэкапа: 3-2-1
# 3 копии данных, 2 разных носителя, 1 за пределами площадки
# Пример скрипта ротации
find /backup -name "*.tar.gz" -mtime +30 -delete       # удалить старше 30 дней
find /backup -name "*.tar.gz" -mtime +7 ! -name "*weekly*" -delete  # оставить недельные
```

> ⚠️ **Антипаттерн:** Бэкап без проверки восстановления. Регулярно тестируй восстановление в изолированной среде. Дополнительный антипаттерн — `rsync` без `--checksum` при важных данных: по умолчанию rsync проверяет только размер и timestamp, а не содержимое.

> ✅ **Senior-совет:** `rsync --link-dest` создаёт инкрементальные бэкапы с hard links — каждый снэпшот выглядит как полный, но занимает только изменённые файлы. 30 ежедневных снэпшотов могут занимать чуть больше одного полного бэкапа.

---

## 6. Планировщики задач

Надёжные планировщики — основа автоматизации production. Systemd timers предпочтительнее cron в современных системах: лучше логирование, зависимости, обработка пропущенных запусков.

```bash
# cron — синтаксис
# ┌─── минута (0-59)
# │ ┌─── час (0-23)
# │ │ ┌─── день месяца (1-31)
# │ │ │ ┌─── месяц (1-12)
# │ │ │ │ ┌─── день недели (0-7, 0=вс)
# │ │ │ │ │
# * * * * * command

crontab -e                                             # редактировать cron текущего пользователя
crontab -l                                             # просмотр
crontab -l -u username                                 # cron другого пользователя
🔐 crontab -e -u www-data                              # редактировать cron www-data

# Отладка cron
grep CRON /var/log/syslog | tail -30                   # Debian/Ubuntu
grep CRON /var/log/cron | tail -30                     # RHEL/CentOS
journalctl -u cron --since "1 hour ago"                # systemd-систему

# Логирование в cron-задаче
*/5 * * * * /usr/bin/python3 /opt/script.py >> /var/log/myjob.log 2>&1
# Или через logger (пишет в syslog/journald):
*/5 * * * * /opt/script.sh 2>&1 | logger -t myjob

# anacron — для задач, которые могут пропускаться
cat /etc/anacrontab
# period  delay  job-identifier  command
# 1       5      daily.backup    /opt/backup.sh
```

```ini
# systemd timer — пример unit-файлов
# /etc/systemd/system/backup.service
[Unit]
Description=Daily Backup
After=network.target

[Service]
Type=oneshot
User=backup
ExecStart=/opt/backup.sh
StandardOutput=journal
StandardError=journal
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily at 2:00

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true          # запустить если пропустили из-за простоя
RandomizedDelaySec=600   # случайная задержка до 10 мин (антишторм)
Unit=backup.service

[Install]
WantedBy=timers.target
```

```bash
# Управление systemd timers
🔐 systemctl enable --now backup.timer
systemctl list-timers --all                            # все таймеры и следующий запуск
systemctl status backup.timer
journalctl -u backup.service --since "7 days ago"      # логи задачи

# Проверка синтаксиса онкалендарь
systemd-analyze calendar "*-*-* 02:00:00"
systemd-analyze calendar "Mon *-*-* 03:00:00"         # по понедельникам
```

> ⚠️ **Антипаттерн:** Cron без перенаправления вывода и без ограничений окружения. Без `2>&1` ошибки уходят на mail root. Без `PATH=` в crontab скрипты падают потому что PATH в cron отличается от интерактивного шелла.

> ✅ **Senior-совет:** Добавь `MAILTO=""` в начало crontab чтобы заглушить email, но при этом пиши логи явно. Для критичных задач используй `flock -n /var/lock/myjob.lock` для предотвращения параллельных запусков: `*/5 * * * * flock -n /tmp/job.lock /opt/job.sh`.

---

## 7. Управление пакетами

Контроль версий пакетов и зависимостей критичен для воспроизводимости сред. Умение работать с репозиториями вручную спасает при инцидентах, когда стандартные инструменты недоступны.

```bash
# apt (Debian/Ubuntu)
apt update && apt upgrade -y                           # обновление
apt install nginx=1.18.0-0ubuntu1                      # конкретная версия
apt-mark hold nginx                                    # заморозить версию
apt-mark unhold nginx                                  # разморозить
apt-mark showhold                                      # что заморожено
apt list --installed | grep nginx                      # установленные nginx
apt show nginx                                         # информация о пакете
apt depends nginx                                      # зависимости
apt rdepends nginx                                     # что зависит от nginx
apt-cache policy nginx                                 # доступные версии
apt autoremove --purge                                 # удалить ненужные + конфиги
dpkg -l | grep nginx                                   # статус через dpkg
dpkg -L nginx                                          # файлы пакета
dpkg -S /usr/sbin/nginx                                # какой пакет содержит файл
dpkg --get-selections > packages.txt                   # список всех пакетов
dpkg --set-selections < packages.txt && apt-get dselect-upgrade  # восстановить

# yum/dnf (RHEL/CentOS/Rocky)
dnf update                                             # обновление
dnf install nginx-1.20.1                               # конкретная версия
dnf versionlock add nginx                              # заморозить версию (нужен плагин)
dnf history                                            # история операций
dnf history undo last                                  # откатить последнюю операцию
dnf info nginx                                         # информация
dnf deplist nginx                                      # зависимости
dnf provides /usr/sbin/nginx                           # какой пакет содержит файл
rpm -qa | grep nginx                                   # установленные
rpm -ql nginx                                          # файлы пакета
rpm -qf /usr/sbin/nginx                                # пакет файла
rpm -V nginx                                           # проверить целостность пакета

# Управление репозиториями
# Debian/Ubuntu
ls /etc/apt/sources.list.d/
add-apt-repository ppa:nginx/stable                    # добавить PPA
apt-key list                                           # ключи репозиториев

# RHEL/CentOS/Rocky
ls /etc/yum.repos.d/
dnf config-manager --add-repo=https://nginx.org/packages/centos/8/x86_64/
dnf repolist all                                       # все репозитории
dnf repoinfo baseos                                    # информация о репозитории

# Сборка из исходников
apt install build-essential                            # Debian/Ubuntu
dnf groupinstall "Development Tools"                   # RHEL
./configure --prefix=/usr/local --with-http_ssl_module
make -j$(nproc)                                        # использовать все ядра
🔐 make install
```

> ⚠️ **Антипаттерн:** `apt upgrade -y` на production без тестирования. Всегда проверяй список изменений (`apt list --upgradable`), тестируй в staging, и используй `apt-mark hold` для критичных компонентов.

> ✅ **Senior-совет:** `rpm -V $(rpm -qf /usr/sbin/sshd)` — проверяет, что файлы пакета не изменены с момента установки. Аналог для Debian: `debsums -c openssh-server`. Незаменимо при подозрении на компрометацию системы.

---

## 8. Сетевые инструменты

Сетевая диагностика — один из самых востребованных навыков в production. Умение анализировать трафик tcpdump и интерпретировать TCP-состояния сокращает время диагностики с часов до минут.

```bash
# ip — современная замена ifconfig
ip addr show                                           # все интерфейсы и адреса
ip addr show eth0                                      # конкретный интерфейс
ip route show                                          # таблица маршрутизации
ip route get 8.8.8.8                                   # маршрут до адреса
ip neigh show                                          # ARP-таблица
ip link show                                           # статус интерфейсов
🔐 ip addr add 10.0.0.10/24 dev eth0                   # добавить адрес
🔐 ip route add 192.168.0.0/24 via 10.0.0.1            # добавить маршрут

# ss — замена netstat (быстрее)
ss -tuln                                               # прослушивающие TCP/UDP порты
ss -tulnp                                              # с именами процессов
ss -s                                                  # статистика
ss -t state established                                # только установленные соединения
ss -t state time-wait | wc -l                          # количество TIME_WAIT
ss -o state established '( dport = :443 or sport = :443 )'  # только HTTPS
ss -tnp dst 10.0.0.1                                   # соединения к конкретному IP
ss -tlnp | grep :8080                                  # кто слушает 8080

# netstat (если ss недоступен)
netstat -tulnp                                         # порты с процессами
netstat -an | grep :80 | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# nmap — сканирование портов
nmap -sV 192.168.1.1                                   # с версиями сервисов
nmap -p 22,80,443 192.168.1.0/24                       # подсеть, конкретные порты
nmap -sn 192.168.1.0/24                                # ping sweep (только хосты)
nmap --open -p 0-1024 192.168.1.1                      # только открытые

# curl — HTTP диагностика
curl -v https://example.com/api/health                 # verbose (заголовки + тело)
curl -I https://example.com                            # только заголовки
curl -w "\n%{http_code} %{time_total}s\n" -o /dev/null https://example.com  # код + время
curl -H "Host: example.com" http://10.0.0.5/           # с виртуальным хостом
curl --resolve example.com:443:10.0.0.5 https://example.com  # с DNS override
curl -k https://example.com                            # игнорировать SSL
curl --max-time 5 http://example.com                   # таймаут 5 секунд
curl -x http://proxy:3128 https://example.com          # через прокси

# dig / nslookup
dig example.com A                                      # A-запись
dig example.com MX                                     # MX-записи
dig @8.8.8.8 example.com                               # через конкретный DNS
dig +trace example.com                                 # трассировка DNS
dig -x 8.8.8.8                                         # reverse DNS
nslookup -type=TXT example.com                         # TXT записи

# traceroute / mtr
traceroute -n 8.8.8.8                                  # без DNS-разрешения (быстрее)
mtr --report --report-cycles 20 8.8.8.8                # статистика за 20 циклов
mtr -n --report 8.8.8.8                                # без DNS в report режиме

# tcpdump — анализ трафика
🔐 tcpdump -i eth0 -n port 80                          # HTTP трафик
🔐 tcpdump -i any -n 'host 10.0.0.1 and port 5432'    # к БД PostgreSQL
🔐 tcpdump -i eth0 -n -w /tmp/capture.pcap             # запись в файл
🔐 tcpdump -r /tmp/capture.pcap -n 'tcp[tcpflags] & tcp-rst != 0'  # RST пакеты
🔐 tcpdump -i eth0 -n 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0' -c 1000  # SYN/FIN
🔐 tcpdump -i eth0 -nn -X -s 0 'port 8080' | head -100  # с payload в hex

# tshark (командный wireshark)
🔐 tshark -i eth0 -f "port 80" -T fields -e ip.src -e http.request.uri 2>/dev/null
```

> ⚠️ **Антипаттерн:** `netstat -an` вместо `ss -s` для получения общей статистики. ss на порядок быстрее и актуальнее. Также антипаттерн — делать `tcpdump` без `-n` на production: DNS-разрешение замедляет захват и может пропустить пакеты.

> ✅ **Senior-совет:** `curl -w "@curl-format.txt"` с кастомным format-файлом даёт детальное время каждой фазы запроса (DNS, connect, TLS handshake, TTFB, total). Незаменимо для диагностики медленных HTTP запросов.

---

## 9. Firewall и безопасность сети

Правильная настройка firewall — первая линия обороны. Важно уметь быстро добавлять временные правила при инциденте и восстанавливать конфигурацию из бэкапа.

```bash
# iptables
🔐 iptables -L -n -v --line-numbers                    # все правила с номерами
🔐 iptables -L INPUT -n -v                             # цепочка INPUT
🔐 iptables -A INPUT -p tcp --dport 22 -j ACCEPT       # разрешить SSH
🔐 iptables -I INPUT 1 -s 10.0.0.0/8 -j ACCEPT        # вставить правило первым
🔐 iptables -D INPUT 5                                 # удалить правило #5
🔐 iptables -A INPUT -j LOG --log-prefix "DROPPED: "   # логировать отброшенные
🔐 iptables -P INPUT DROP                              # политика по умолчанию DROP
🔐 iptables -F                                         # 💣 очистить все правила!

# NAT с iptables
🔐 iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE  # NAT для интерфейса
🔐 iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 10.0.0.5:8080

# Сохранение и восстановление
🔐 iptables-save > /etc/iptables/rules.v4              # Debian/Ubuntu
🔐 iptables-restore < /etc/iptables/rules.v4
🔐 service iptables save                               # RHEL/CentOS
🔐 iptables-save | grep -v "^#" | grep -c "^"         # подсчёт правил

# nftables (современная замена iptables)
🔐 nft list ruleset                                    # все правила
🔐 nft add rule inet filter input tcp dport 80 accept  # разрешить HTTP
🔐 nft -f /etc/nftables.conf                           # загрузить конфиг

# ufw (Debian/Ubuntu — простой интерфейс)
🔐 ufw status verbose                                  # статус
🔐 ufw allow 22/tcp                                    # разрешить SSH
🔐 ufw allow from 10.0.0.0/8 to any port 5432          # PostgreSQL из подсети
🔐 ufw deny 3306                                       # запретить MySQL
🔐 ufw delete allow 80/tcp                             # удалить правило
🔐 ufw enable                                          # включить
🔐 ufw logging on                                      # логирование

# firewalld (RHEL/CentOS)
🔐 firewall-cmd --list-all                             # текущие правила
🔐 firewall-cmd --add-port=8080/tcp --permanent        # добавить порт навсегда
🔐 firewall-cmd --reload                               # применить изменения
🔐 firewall-cmd --zone=trusted --add-source=10.0.0.0/8 --permanent
🔐 firewall-cmd --direct --add-rule ipv4 filter INPUT 0 -p tcp --dport 80 -j ACCEPT
```

> ⚠️ **Антипаттерн:** Изменять правила iptables на production без возможности отката. Всегда делай `iptables-save > /tmp/rules_backup_$(date +%H%M).txt` перед изменениями. При работе по SSH — устанавливай таймер автооткатки: `(sleep 120 && iptables-restore < /tmp/backup.txt) &`.

> ✅ **Senior-совет:** `watch -n 1 iptables -nvL INPUT --line-numbers` — мониторинг счётчиков в реальном времени. Помогает убедиться, что правило вообще матчит трафик и выявить неожиданные источники соединений.

---

## 10. SSH и удалённый доступ

SSH с правильной конфигурацией — ключ к безопасному управлению инфраструктурой. Jump hosts и multiplexing значительно ускоряют работу в сложных топологиях.

```ini
# ~/.ssh/config — пример production конфигурации
Host bastion
    HostName bastion.company.com
    User admin
    IdentityFile ~/.ssh/bastion_key
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host prod-*
    User deploy
    IdentityFile ~/.ssh/prod_key
    ProxyJump bastion           # jump через bastion
    StrictHostKeyChecking yes
    UserKnownHostsFile ~/.ssh/known_hosts.prod

Host dev-*
    User developer
    ProxyJump bastion
    StrictHostKeyChecking no    # только для dev!

Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm_%r@%h:%p    # мультиплексирование
    ControlPersist 10m
    Compression yes
    ConnectTimeout 10
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

```bash
# SSH ключи
ssh-keygen -t ed25519 -C "admin@company.com" -f ~/.ssh/prod_key
ssh-keygen -t rsa -b 4096 -C "legacy system" -f ~/.ssh/legacy_key
ssh-copy-id -i ~/.ssh/prod_key.pub user@server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys    # вручную

# SSH Agent
eval $(ssh-agent)                                      # запустить agent
ssh-add ~/.ssh/prod_key                                # добавить ключ
ssh-add -l                                             # список ключей в агенте
ssh-add -t 3600 ~/.ssh/key                             # ключ на 1 час
ssh-add -D                                             # удалить все ключи

# Port forwarding
ssh -L 5432:db-server:5432 bastion                     # local forward (PostgreSQL через bastion)
ssh -L 8080:localhost:8080 prod-server                 # локальный тоннель к сервисе
ssh -R 9090:localhost:9090 public-server               # remote forward (reverse tunnel)
ssh -D 1080 bastion                                    # dynamic (SOCKS proxy)
ssh -N -f -L 5432:db:5432 bastion                      # фоновый forward без команды

# Копирование через jump host
scp -J bastion file.txt prod-app:/tmp/

# Проверка соединения / отладка
ssh -v user@server 2>&1 | head -50                     # verbose подключение
ssh -vvv user@server 2>&1 | grep "Authentications"    # debug аутентификации
ssh -o BatchMode=yes user@server echo ok               # проверка без пароля

# Hardening sshd_config (ключевые параметры)
grep -E "PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|AllowUsers|Port" /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config — hardening
Port 22
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PermitEmptyPasswords no
X11Forwarding no
AllowUsers deploy admin
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
# Разрешить только современные алгоритмы:
KexAlgorithms curve25519-sha256,ecdh-sha2-nistp256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
```

```bash
# Проверка конфига и перезагрузка
🔐 sshd -t                                             # проверить синтаксис
🔐 sshd -T | grep -i "permit\|password\|pubkey"       # дамп эффективных настроек
🔐 systemctl reload sshd                               # reload (не restart!)
```

> ⚠️ **Антипаттерн:** `systemctl restart sshd` при активных SSH-сессиях — может разорвать соединения. Всегда используй `systemctl reload sshd`. Ещё антипаттерн — отключать SSH на порту 22 без проверки работы нового порта в отдельной сессии.

> ✅ **Senior-совет:** `ControlMaster` + `ControlPersist 10m` в `~/.ssh/config` мультиплексирует SSH-соединения. Повторные подключения к тому же хосту используют существующий тоннель — вход мгновенный, без повторной аутентификации. Экономит секунды на каждом подключении при интенсивной работе.

---

## 11. Systemd и управление сервисами

Systemd — стандарт управления сервисами в modern Linux. Глубокое понимание unit-файлов и journald позволяет строить надёжные, самовосстанавливающиеся сервисы.

```bash
# systemctl — базовые операции
systemctl status nginx                                 # статус сервиса
systemctl start/stop/restart/reload nginx              # управление
systemctl enable/disable nginx                         # автозапуск
systemctl is-active/is-enabled nginx                   # проверка состояния
systemctl list-units --type=service                    # все активные сервисы
systemctl list-units --type=service --state=failed     # упавшие сервисы
systemctl list-unit-files --type=service               # все (включая неактивные)

# Зависимости и цепочки
systemctl list-dependencies nginx                      # что нужно nginx
systemctl list-dependencies --reverse nginx            # что зависит от nginx
systemctl cat nginx                                    # показать unit-файл

# Override (без изменения оригинала)
🔐 systemctl edit nginx                                # создать override
🔐 systemctl edit --full nginx                         # полная копия unit-файла
ls /etc/systemd/system/nginx.service.d/                # override файлы
```

```ini
# /etc/systemd/system/myapp.service — production unit-файл
[Unit]
Description=My Production Application
After=network.target postgresql.service
Requires=postgresql.service
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

# Лимиты ресурсов
LimitNOFILE=65536
LimitNPROC=512
MemoryMax=1G
CPUQuota=200%

# Безопасность
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp /var/log/myapp
PrivateTmp=yes

# Логирование
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
```

```bash
# journalctl — работа с логами
journalctl -u nginx                                    # логи сервиса
journalctl -u nginx --since "1 hour ago"               # за последний час
journalctl -u nginx -n 100 --no-pager                  # последние 100 строк
journalctl -u nginx -f                                 # follow (как tail -f)
journalctl -p err -u nginx                             # только ошибки
journalctl --since "2025-01-15 10:00:00"               # с конкретного времени
journalctl -b                                          # с текущей загрузки
journalctl -b -1                                       # с предыдущей загрузки
journalctl -k                                          # только kernel messages
journalctl --disk-usage                                # размер журнала
🔐 journalctl --vacuum-time=30d                        # очистить старше 30 дней
journalctl -o json-pretty -u nginx | head              # в JSON формате
journalctl -u myapp --grep="ERROR|CRITICAL"            # поиск по тексту

# Targets (уровни запуска)
systemctl get-default                                  # текущий target
🔐 systemctl set-default multi-user.target             # консоль без GUI
🔐 systemctl isolate rescue.target                     # аварийный режим

# Socket activation
systemctl list-sockets                                 # активированные сокеты
```

> ⚠️ **Антипаттерн:** Редактировать файлы в `/lib/systemd/system/` напрямую — они перезаписываются при обновлении пакетов. Всегда используй `/etc/systemd/system/` или `systemctl edit`. После любых изменений в unit-файлах обязательно `systemctl daemon-reload`.

> ✅ **Senior-совет:** `systemd-analyze blame` показывает время старта каждого сервиса при загрузке, `systemd-analyze critical-chain` — критический путь. После добавления нового сервиса всегда запускай оба, чтобы убедиться что новый сервис не стал bottleneck при загрузке.

---

## 12. Мониторинг ресурсов

Правильная интерпретация метрик — разница между паникой и системным анализом. Понимание взаимосвязей между CPU, памятью, I/O и сетью даёт полную картину состояния системы.

```bash
# vmstat — память, swap, CPU, I/O
vmstat 1 10                                            # каждую секунду, 10 раз
vmstat -s                                              # сводная статистика
# Столбцы: r=runqueue, b=blocked, si/so=swap_in/out, bi/bo=disk_in/out
# wa=iowait%, id=idle%, us=user%, sy=system%, st=stolen%

# iostat — дисковая I/O
iostat -xz 1 5                                         # расширенная, каждую сек, 5 раз
iostat -dh                                             # только диски, human-readable
# Ключевые метрики: %util, await (мс ожидания), r/s, w/s, rMB/s, wMB/s

# sar — системная активность (нужен sysstat)
sar -u 1 10                                            # CPU 10 раз с интервалом 1с
sar -r 1 5                                             # память
sar -b 1 5                                             # I/O
sar -n DEV 1 5                                         # сеть по интерфейсам
sar -f /var/log/sysstat/sa15                           # исторические данные (15-е число)
sar -A                                                 # всё сразу

# free — память
free -h                                                # human-readable
free -h -s 2                                           # обновлять каждые 2 сек
# available = реально свободно для приложений (с учётом cache)

# lsof — открытые файлы
lsof -p 1234                                           # файлы процесса
lsof -u www-data                                       # файлы пользователя
lsof -i :80                                            # процессы на порту 80
lsof -i tcp:1-1024                                     # привилегированные порты
lsof +D /var/log                                       # кто открыл файлы в директории
lsof | wc -l                                           # всего открытых файловых дескрипторов
lsof | grep "(deleted)" | awk '{print $1, $2, $7}' | sort -k3 -rh  # удалённые но открытые

# strace — системные вызовы
strace -p 1234                                         # attach к процессу
strace -p 1234 -e trace=network                        # только сетевые вызовы
strace -p 1234 -e trace=file                           # файловые вызовы
strace -c -p 1234                                      # статистика вызовов
strace -tt -T -p 1234 2>&1 | head -50                  # с временными метками и длительностью
strace -o /tmp/trace.log /usr/sbin/nginx -t            # запуск с трассировкой

# dstat — интегрированный мониторинг
dstat -cdngy                                           # cpu, disk, net, page, sys
dstat -D sda,sdb                                       # конкретные диски
dstat --top-cpu --top-io --top-mem                     # топ процессов
```

> ⚠️ **Антипаттерн:** Смотреть на `%CPU` в `top` при анализе нагрузки. `load average` и `r` (runqueue) в `vmstat` важнее: они показывают сколько процессов ждут CPU. Load average > количества ядер CPU = проблема.

> ✅ **Senior-совет:** `strace -c -p PID` в течение 30 секунд показывает, какие системные вызовы тратят больше всего времени. Завис ли процесс на `futex` (mutex contention), на `read` (I/O wait), или на `select/epoll` (ждёт сеть) — сразу понятно направление диагностики.

---

## 13. Дисковая подсистема

Управление дисками и LVM в production требует точности — ошибки необратимы. LVM позволяет расширять тома онлайн, что критично для непрерывной работы сервисов.

```bash
# Информация о блочных устройствах
lsblk -f                                               # диски с ФС и UUID
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID
fdisk -l                                               # детальная информация
parted -l                                              # GPT/MBR, партиции
blkid                                                  # UUID и типы ФС

# fdisk — MBR партиции
🔐 fdisk /dev/sdb                                      # интерактивно
🔐 echo -e "n\np\n1\n\n\nw" | fdisk /dev/sdb           # создать партицию автоматически

# parted — GPT
🔐 parted /dev/sdb mklabel gpt
🔐 parted /dev/sdb mkpart primary ext4 0% 100%
🔐 parted /dev/sdb print

# mkfs — создание ФС
🔐 mkfs.ext4 -L "data" /dev/sdb1
🔐 mkfs.xfs /dev/sdb1                                  # XFS (предпочтительна для RHEL)
🔐 mkfs.ext4 -b 4096 -m 0 /dev/sdb1                    # без reserved blocks (для данных)

# mount / fstab
🔐 mount /dev/sdb1 /mnt/data
🔐 mount -t xfs -o noatime,nodiratime /dev/sdb1 /mnt   # с опциями
mount | column -t                                      # текущие маунты
🔐 umount -l /mnt/data                                 # lazy unmount (не ждёт)

# /etc/fstab
# UUID=xxx /data ext4 defaults,noatime,errors=remount-ro 0 2
🔐 mount -a                                            # применить fstab
🔐 mount -a -fv                                        # dry run проверки fstab

# LVM
🔐 pvcreate /dev/sdb1                                  # создать Physical Volume
🔐 vgcreate data-vg /dev/sdb1                          # Volume Group
🔐 lvcreate -L 10G -n app-lv data-vg                   # Logical Volume 10GB
🔐 lvcreate -l 100%FREE -n app-lv data-vg              # всё свободное место
pvs; vgs; lvs                                          # статус LVM
pvdisplay; vgdisplay; lvdisplay                        # детальная информация

# Расширение LVM онлайн (без остановки сервиса!)
🔐 pvresize /dev/sdb1                                  # после расширения диска
🔐 lvextend -L +10G /dev/data-vg/app-lv               # расширить LV
🔐 lvextend -l +100%FREE /dev/data-vg/app-lv           # добавить всё свободное
🔐 resize2fs /dev/data-vg/app-lv                       # ext4 — расширить ФС
🔐 xfs_growfs /mnt/data                                # XFS — расширить ФС

# RAID (mdadm)
🔐 mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat                                       # состояние RAID
🔐 mdadm --detail /dev/md0                             # детальная информация
🔐 mdadm /dev/md0 --add /dev/sdd                       # добавить диск (rebuild)
🔐 mdadm /dev/md0 --fail /dev/sdc --remove /dev/sdc   # вывести диск

# Проверка и восстановление ФС
🔐 fsck -n /dev/sdb1                                   # проверить без исправления
🔐 e2fsck -f /dev/sdb1                                 # полная проверка ext4
🔐 xfs_repair /dev/sdb1                                # ремонт XFS
tune2fs -l /dev/sdb1                                   # параметры ext4
dumpe2fs /dev/sdb1 | grep -i "error\|free\|block"      # информация об ошибках
```

> ⚠️ **Антипаттерн:** `fsck` на смонтированной файловой системе (кроме live-recovery режима). При необходимости проверки используй read-only mount или загружайся с live-образа.

> ✅ **Senior-совет:** `tune2fs -e remount-ro /dev/sdb1` настраивает автоматический переход в read-only при ошибках ФС — лучше read-only чем corruption. Для XFS аналогично через `xfs_admin`. Добавь `errors=remount-ro` в fstab для критичных ФС.

---

## 14. Ядро и система

Параметры ядра существенно влияют на производительность и стабильность. Некоторые настройки критичны для высоконагруженных production-систем.

```bash
# Информация о системе
uname -r                                               # версия ядра
uname -a                                               # полная информация
cat /etc/os-release                                    # версия дистрибутива
hostnamectl                                            # hostname и OS информация
uptime                                                 # аптайм и load average
last reboot                                            # история перезагрузок

# dmesg — сообщения ядра
dmesg -T | tail -50                                    # с человекочитаемым временем
dmesg -T --level=err,crit                              # только ошибки
dmesg -T -w                                            # follow (как tail -f)
dmesg | grep -i "oom\|kill\|error\|fail" | tail -30    # проблемные сообщения
dmesg | grep -i "hardware error\|mce"                  # аппаратные ошибки

# sysctl — параметры ядра
sysctl -a | grep net.ipv4                              # сетевые параметры
sysctl -a | grep vm.swappiness                         # swappiness
🔐 sysctl -w vm.swappiness=10                          # изменить временно
🔐 sysctl -p /etc/sysctl.conf                          # применить из файла

# /proc — виртуальная ФС ядра
cat /proc/cpuinfo | grep "model name" | head -1        # модель CPU
grep MemAvailable /proc/meminfo                        # доступная память
cat /proc/loadavg                                      # load average
cat /proc/sys/fs/file-nr                               # открытые/макс файловые дескрипторы
cat /proc/net/sockstat                                 # статистика сокетов
cat /proc/1234/cmdline | tr '\0' ' '                   # командная строка процесса
ls -la /proc/1234/fd | wc -l                           # файловые дескрипторы процесса
cat /proc/1234/status | grep -i "vmrss\|vmswap\|threads"

# Модули ядра
lsmod                                                  # загруженные модули
modinfo tcp_bbr                                        # информация о модуле
🔐 modprobe tcp_bbr                                    # загрузить модуль
🔐 modprobe -r module_name                             # выгрузить модуль
echo "tcp_bbr" >> /etc/modules-load.d/bbr.conf         # автозагрузка
```

```bash
# Важные production sysctl параметры
# /etc/sysctl.d/99-production.conf

# Сеть
net.core.somaxconn = 65535           # backlog входящих соединений
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_tw_reuse = 1            # повторно использовать TIME_WAIT сокеты
net.ipv4.ip_local_port_range = 1024 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_congestion_control = bbr  # BBR congestion control

# Память
vm.swappiness = 10                   # меньше использовать swap (0-10 для серверов)
vm.dirty_ratio = 15                  # % RAM до принудительного flush на диск
vm.dirty_background_ratio = 5        # % RAM при котором начинается фоновый flush
vm.overcommit_memory = 1             # для Redis/некоторых БД

# Файловые дескрипторы
fs.file-max = 2097152
fs.nr_open = 2097152

# Hugepages (для СУБД)
vm.nr_hugepages = 1024               # 1024 * 2MB = 2GB huge pages
```

> ⚠️ **Антипаттерн:** `sysctl -w` изменения без записи в `/etc/sysctl.d/` — не переживут перезагрузку. Также антипаттерн — `vm.overcommit_memory = 1` без понимания последствий: при нехватке памяти OOM Killer будет убивать процессы непредсказуемо.

> ✅ **Senior-совет:** `sysctl -a 2>/dev/null | grep -v "^#" > /tmp/sysctl_before.txt` перед внесением изменений и аналогично после — быстрый diff покажет что изменилось. Бесценно при разборе инцидентов "что изменилось в ядре".

---

## 15. TROUBLESHOOTING

> 🚨 **Методология прежде всего:** Инцидент — это не время для угадывания. Следуй методу: **Симптом → Данные → Гипотеза → Проверка → Решение → Постмортем**.

---

### 15.1 Методология диагностики

**USE Method** (для ресурсов):
- **U**tilization — насколько ресурс занят (%)
- **S**aturation — очередь/задержка ожидания ресурса
- **E**rrors — ошибки на уровне ресурса

**RED Method** (для сервисов):
- **R**ate — запросы в секунду
- **E**rrors — процент ошибок
- **D**uration — время ответа (латентность)

```bash
# 🔍 Первые команды при инциденте — первые 60 секунд
uptime                                                 # load average тренд
dmesg -T | tail -20                                    # свежие сообщения ядра
journalctl -p err --since "5 minutes ago"              # ошибки системы
systemctl list-units --state=failed                    # упавшие сервисы
df -h                                                  # диск не полный?
free -h                                                # память не кончилась?
ps aux --sort=-%cpu | head -10                         # CPU-грузящие процессы
ps aux --sort=-%mem | head -10                         # память-грузящие процессы
ss -s                                                  # статистика сокетов
netstat -an 2>/dev/null | awk '{print $6}' | sort | uniq -c | sort -rn  # TCP состояния
```

---

### 15.2 Диагностика CPU

#### Симптом
Load average неожиданно высокий, система медленно реагирует, приложение деградирует.

#### Диагностика

```bash
# Различие load average и CPU usage
uptime                                                 # load average (1/5/15 мин)
nproc                                                  # количество vCPU
# load > nproc → очередь к CPU или I/O ожидание

# Разбить load по CPU vs I/O
vmstat 1 5
# r — ожидают CPU, b — ожидают I/O
# wa > 10% → I/O проблема, не CPU!

# Поиск процессов-виновников
ps aux --sort=-%cpu | head -15
top -b -n 1 | head -20                                 # snapshot без интерактивности
pidstat -u 1 5                                         # активность по процессам

# CPU steal time в виртуальных средах
vmstat 1 5 | awk '{print $16}'                         # столбец 'st' — steal%
# steal > 5% → конкуренция на гипервизоре, нужно обращаться к провайдеру

# CPU throttling в контейнерах (K8s/Docker)
cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled       # cgroups v1
cat /sys/fs/cgroup/cpu.stat | grep throttled           # cgroups v2
# throttled_usec высокий → CPU limit слишком мал

# Профилирование через perf
🔐 perf top -p 1234                                    # hot functions в реальном времени
🔐 perf record -g -p 1234 sleep 30                     # запись 30 сек с callgraph
🔐 perf report --stdio | head -50                      # отчёт в текстовом виде
# Flamegraph (brendangregg/FlameGraph)
🔐 perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > cpu.svg

# Конкретные ядра
mpstat -P ALL 1 5                                      # нагрузка по каждому ядру
# Одно ядро 100% при низком общем → single-threaded bottleneck
```

#### Решение

```bash
# Временно снизить нагрузку
renice -n 19 -p $(pgrep hungry_process)                # снизить приоритет
cpulimit -p 1234 -l 50                                 # ограничить до 50% (нужен cpulimit)
🔐 systemctl set-property myservice CPUQuota=100%       # через systemd

# Долгосрочно: анализ кода, горизонтальное масштабирование, настройка BBR
```

#### Профилактика

```bash
# Мониторинг load average / количество ядер
echo "Load ratio: $(uptime | awk '{print $NF}') / $(nproc)"
# Алерт если load > nproc * 2 в течение >5 минут
```

---

### 15.3 Диагностика памяти

#### Симптом
OOM Killer убивает процессы, система уходит в своп, приложение работает медленно или падает.

#### Диагностика

```bash
# 🔍 Первые команды при инциденте с памятью
free -h
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapUsed|Cached|Buffers"
vmstat 1 5                                             # si/so > 0 → активный swap

# OOM Killer — найти жертв
dmesg -T | grep -i "oom\|killed\|out of memory" | tail -30
journalctl -k --since "1 hour ago" | grep -i oom
# Вывод OOM Killer содержит:
# - причину (процесс который исчерпал память)
# - score каждого процесса (oom_score)
# - итоговую жертву

# OOM score процессов
for pid in $(ls /proc | grep "^[0-9]"); do
  oom=$(cat /proc/$pid/oom_score 2>/dev/null)
  comm=$(cat /proc/$pid/comm 2>/dev/null)
  [ -n "$oom" ] && echo "$oom $pid $comm"
done | sort -rn | head -20                             # кандидаты на убийство

# Анализ потребления памяти
smem -tk 2>/dev/null | tail -20                        # PSS (реальное потребление)
ps aux --sort=-%mem | head -20
# RSS vs VSZ: RSS — реально в RAM, VSZ — виртуальное (включает swap, mmap)

# Swap анализ
swapon --show                                          # swap устройства
cat /proc/swaps
# Если много в swap при наличии RAM → vm.swappiness слишком высокий

# Утечки памяти — слежение за ростом
watch -n 5 'ps -o pid,comm,rss --sort=-rss | head -10'

# Hugepages
grep -i huge /proc/meminfo                             # использование hugepages
cat /proc/sys/vm/nr_hugepages

# valgrind (для диагностики утечек в DEV)
valgrind --leak-check=full --track-origins=yes ./myapp
```

#### Решение

```bash
# Немедленное освобождение кэша (с осторожностью!)
🔐 echo 3 > /proc/sys/vm/drop_caches                  # 💣 сбросить pagecache+dentries+inodes

# Защита критичного процесса от OOM
echo -1000 > /proc/$(pgrep nginx)/oom_score_adj        # 🔐 никогда не убивать nginx
echo 500 > /proc/$(pgrep dbcleaner)/oom_score_adj      # убивать первым

# Настройка OOM поведения
🔐 sysctl -w vm.panic_on_oom=0                         # OOM Killer а не паника ядра
🔐 sysctl -w vm.oom_kill_allocating_task=1             # убивать виновника а не случайного
```

#### Профилактика

```bash
# Мониторинг MemAvailable (не MemFree!)
awk '/MemAvailable/ {print $2}' /proc/meminfo          # в KB
# Алерт если MemAvailable < 10% от MemTotal
```

---

### 15.4 Диагностика диска и I/O

#### Симптом
Высокий iowait, медленное приложение, диск 100% утилизации, ошибки записи, нет места на диске.

#### Диагностика

```bash
# 🔍 Первые команды при инциденте с диском
df -h                                                  # свободное место
df -i                                                  # inode!
iostat -xz 1 5                                         # I/O статистика

# Интерпретация iostat:
# %util > 80% → диск перегружен
# await > 20ms (SSD) или > 100ms (HDD) → проблема с задержкой
# avgqu-sz > 1 → очередь к диску

# Поиск виновных процессов I/O
🔐 iotop -oP                                           # процессы с активным I/O
🔐 pidstat -d 1 5                                      # I/O по процессам
ls /proc/*/io 2>/dev/null | head -5                    # пример файла
cat /proc/1234/io                                      # I/O статистика процесса

# Нет места: найти скрытые файлы
df -h /var
du -sh /var/* | sort -rh | head -20
lsof +L1 | awk '{print $7, $9}' | sort -rn | head -10 # удалённые открытые файлы!
# Стандартный кейс: /var/log/*.log переполнен или приложение пишет в /tmp

# Проблемы с inode
df -i                                                  # inode usage
# Если inode 100% при наличии места → слишком много мелких файлов
find /var -maxdepth 3 -type d | while read dir; do
  count=$(ls -1 "$dir" 2>/dev/null | wc -l)
  echo "$count $dir"
done | sort -rn | head -10                             # директории с наибольшим числом файлов

# Диагностика ФС
🔐 tune2fs -l /dev/sda1 | grep -i "error\|check\|mount"
dmesg -T | grep -i "ext4\|xfs\|disk\|error\|io error" | tail -20
smartctl -a /dev/sda | grep -E "Raw_Read_Error|Reallocated|Spin_Retry|SMART"
```

#### Решение

```bash
# Немедленно освободить место
journalctl --vacuum-size=1G                            # обрезать journald
🔐 find /tmp -mtime +1 -delete                         # очистить старые tmp
🔐 find /var/log -name "*.gz" -mtime +30 -delete       # старые архивы логов
# После освобождения — разобраться с причиной

# Ускорение I/O (временно)
🔐 echo mq-deadline > /sys/block/sda/queue/scheduler   # HDD
🔐 echo none > /sys/block/nvme0n1/queue/scheduler      # NVMe/SSD

# Сброс кэша записи (если нужно срочно освободить dirty pages)
🔐 sync; echo 1 > /proc/sys/vm/drop_caches
```

#### Профилактика

```bash
# Мониторинг места с запасом
df -h | awk '$5+0 > 80 {print "WARNING:", $0}'         # алерт при >80%
df -i | awk '$5+0 > 80 {print "INODE WARNING:", $0}'   # алерт inode >80%
```

---

### 15.5 Диагностика сети

#### Симптом
Потери пакетов, высокая латентность, соединения зависают, TIME_WAIT переполнение, DNS не резолвит.

#### Диагностика

```bash
# 🔍 Первые команды при сетевом инциденте
ip addr show                                           # интерфейсы и IP
ip route                                               # маршрутизация
ss -s                                                  # статистика сокетов
cat /proc/net/sockstat                                 # подробная статистика

# Потери пакетов
ping -c 100 -i 0.1 8.8.8.8 | tail -3                  # быстрые 100 пингов
mtr --report -c 50 8.8.8.8                             # трассировка с потерями
# Loss% на каждом хопе → где именно потеря

# TCP состояния — анализ
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# TIME_WAIT много → нормально при высоком трафике, можно снизить tcp_time_wait
# CLOSE_WAIT много → утечка в приложении (не закрывает соединения)
# SYN_RECV много → возможная SYN-flood атака

# TIME_WAIT насыщение
cat /proc/sys/net/ipv4/tcp_tw_reuse                    # включено ли переиспользование
ss -tan state time-wait | wc -l                        # количество TIME_WAIT

# Буферы сокетов переполнены
ss -tnp | awk '{print $2, $3}' | sort -k1 -rn | head # Recv-Q / Send-Q
# Send-Q > 0 → удалённая сторона не успевает принимать
# Recv-Q > 0 → приложение не успевает читать

# DNS диагностика
dig google.com                                         # базовая проверка
dig @8.8.8.8 google.com                                # через внешний DNS
dig @localhost google.com                              # через локальный
resolvectl status 2>/dev/null || cat /etc/resolv.conf  # конфигурация DNS
systemd-resolve --statistics 2>/dev/null               # кэш статистика

# tcpdump для анализа проблемы
🔐 tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-rst != 0' -c 100  # RST пакеты
🔐 tcpdump -i eth0 -n 'port 8080 and (tcp[tcpflags] & tcp-syn != 0)' -c 50  # SYN к 8080
🔐 tcpdump -i eth0 -n 'port 443' -w /tmp/https_$(date +%s).pcap &  # запись и анализ потом

# Скрытые проблемы сети
ethtool -S eth0 | grep -i "error\|drop\|miss"          # ошибки NIC
ip -s link show eth0                                   # TX/RX ошибки
cat /proc/net/dev                                      # счётчики по интерфейсам
```

#### Решение

```bash
# TIME_WAIT — уменьшить таймаут (с осторожностью!)
🔐 sysctl -w net.ipv4.tcp_tw_reuse=1                   # reuse для outbound
🔐 sysctl -w net.ipv4.tcp_fin_timeout=15               # ускорить FIN

# Буферы сокетов
🔐 sysctl -w net.core.rmem_max=134217728
🔐 sysctl -w net.core.wmem_max=134217728

# CLOSE_WAIT — только через исправление приложения
# Проверить какое приложение не закрывает сокеты:
ss -tnp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn
```

#### Профилактика

```bash
# Мониторинг CLOSE_WAIT (признак утечки соединений)
count=$(ss -tan state close-wait | wc -l)
[ "$count" -gt 100 ] && echo "WARNING: $count CLOSE_WAIT connections"
```

---

### 15.6 Диагностика приложений и сервисов

#### Симптом
Процесс завис, не отвечает, аномально высокое потребление ресурсов, ошибки в логах, segfault.

#### Диагностика

```bash
# 🔍 Первые команды при проблеме с сервисом
systemctl status myapp -l                              # статус и последние логи
journalctl -u myapp -n 100 --no-pager                  # логи
journalctl -u myapp --since "10 minutes ago" -p err    # ошибки за 10 мин

# Зависший процесс — что он делает?
🔐 strace -p 1234 -e trace=all -T 2>&1 | head -30      # системные вызовы
# futex → mutex/lock contention
# select/poll/epoll_wait → ждёт событие (нормально для I/O сервисов)
# read без возврата → ждёт данные (возможно сетевой таймаут)

# Файловые дескрипторы
ls /proc/1234/fd | wc -l                               # текущее количество
cat /proc/1234/limits | grep "open files"              # лимит
lsof -p 1234 | wc -l                                   # то же через lsof
lsof -p 1234 | grep -c "sock"                          # сколько сокетов открыто

# Ulimit
ulimit -n                                              # лимит текущей сессии
cat /proc/sys/fs/nr_open                               # системный максимум
# Для systemd сервиса: grep LimitNOFILE /lib/systemd/system/myapp.service

# Сокеты процесса
ss -p | grep "pid=1234"                                # сокеты процесса
lsof -p 1234 -i                                        # только сетевые

# Segfault диагностика
dmesg | grep "segfault\|general protection" | tail -10
journalctl -k | grep segfault | tail -10
# Вывод: myapp[1234]: segfault at addr ip addr sp addr error N in libname.so

# Core dump
🔐 ulimit -c unlimited                                 # включить core dump
cat /proc/sys/kernel/core_pattern                      # куда пишутся cores
# Или через systemd:
coredumpctl list                                       # список core dumps
coredumpctl gdb myapp                                  # анализ в gdb
coredumpctl info 1234                                  # информация о core

# Анализ core dump (без gdb)
file /var/lib/systemd/coredump/core.myapp.*
🔐 gdb /usr/bin/myapp /var/lib/systemd/coredump/core.myapp.xxx
# В gdb: bt (backtrace), info threads, thread apply all bt
```

```bash
# journalctl — продвинутые фильтры
journalctl -u nginx --since "2025-01-15 10:00:00" --until "2025-01-15 11:00:00"
journalctl PRIORITY=3 --since "1 hour ago"             # только err и хуже
journalctl _SYSTEMD_UNIT=nginx.service _PID=1234       # конкретный сервис и PID
journalctl -o verbose -u myapp | head                  # все поля записи
journalctl --grep="timeout|connection refused" -u myapp
```

#### Решение

```bash
# Исчерпание файловых дескрипторов
🔐 systemctl set-property myapp.service LimitNOFILE=65536  # через systemd
# Или в /etc/security/limits.d/myapp.conf:
# myapp soft nofile 65536
# myapp hard nofile 65536

# Перезапуск с graceful (не kill -9)
systemctl reload myapp                                 # если поддерживается
systemctl restart myapp                                # полный перезапуск
```

#### Профилактика

```bash
# Автоматический restart через systemd
# В unit-файле: Restart=on-failure, RestartSec=10s
# Лимит попыток: StartLimitBurst=5, StartLimitIntervalSec=60

# Периодическая проверка file descriptors
watch -n 60 'ls /proc/$(pgrep myapp)/fd | wc -l'
```

---

### 15.7 Диагностика загрузки системы (Boot)

#### Симптом
Медленная загрузка системы, зависание при старте, системный юнит не стартует, ошибки fstab.

#### Диагностика

```bash
# 🔍 Анализ времени загрузки
systemd-analyze                                        # общее время загрузки
systemd-analyze blame | head -20                       # топ медленных сервисов
systemd-analyze critical-chain                         # критический путь загрузки
systemd-analyze plot > boot_$(date +%Y%m%d).svg        # график загрузки

# Проблема с конкретным сервисом при загрузке
systemctl status --full servicename                    # ошибки старта
journalctl -b -u servicename                           # логи данного boot
journalctl -b                                          # все логи текущей загрузки
journalctl -b -1                                       # логи предыдущей загрузки

# Зависшие юниты
systemctl list-units --state=failed
systemctl list-jobs                                    # активные задачи systemd

# Проблемы с fstab
🔐 mount -a -fv                                        # проверка без монтирования
# Если ошибка → закомментировать строку в fstab, перезагрузиться
# Правильное fstab: добавь nofail для несрочных дисков:
# UUID=xxx /backup ext4 defaults,nofail,x-systemd.automount 0 2

# GRUB
🔐 grep -v "^#\|^$" /boot/grub/grub.cfg | head -30    # текущий конфиг
🔐 update-grub                                         # Debian/Ubuntu
🔐 grub2-mkconfig -o /boot/grub2/grub.cfg              # RHEL/CentOS
```

```bash
# Восстановление из GRUB — аварийный boot
# При загрузке нажать 'e' в GRUB, найти строку linux/linuxefi, добавить:
# systemd.unit=rescue.target  — rescue mode (минимальные сервисы)
# systemd.unit=emergency.target  — только ядро, root ФС
# init=/bin/bash  — прямой bash (если systemd сломан)

# Восстановление пароля root
# 1. GRUB → edit → добавить: rw init=/bin/bash или rd.break
# 2. В emergency shell:
🔐 mount -o remount,rw /                               # rw если ro
🔐 passwd root                                         # новый пароль
🔐 exec /sbin/init                                     # или reboot -f

# chroot в повреждённую систему с live-образа
🔐 mount /dev/sda1 /mnt                                # корневая ФС
🔐 mount --bind /dev /mnt/dev
🔐 mount --bind /proc /mnt/proc
🔐 mount --bind /sys /mnt/sys
🔐 mount --bind /run /mnt/run
🔐 chroot /mnt /bin/bash
# Теперь можно: passwd root, grub-install, apt fix-broken, etc.
```

#### Профилактика

```bash
# Всегда добавляй nofail для необязательных дисков в fstab
# Тестируй fstab перед перезагрузкой: mount -a --fake
# После обновления grub — проверь конфиг: grub-script-check /boot/grub/grub.cfg
```

---

### 15.8 Диагностика безопасности

#### Симптом
Подозрительная активность, неизвестные процессы, неожиданный исходящий трафик, изменение системных файлов.

#### Диагностика

```bash
# 🔍 Первые команды при подозрении на компрометацию
w                                                      # текущие пользователи
who                                                    # кто залогинен
last | head -30                                        # история входов
lastlog                                                # последний вход всех пользователей
last -F | grep -v "^reboot\|^shutdown" | head -20     # только логины пользователей

# Необычные процессы
ps aux --forest                                        # дерево процессов
ps aux | grep -v "^root\|^www-data\|^mysql\|^nobody"  # нестандартные пользователи
lsof -i | grep ESTABLISHED                             # активные соединения
ss -tupn                                               # все соединения с PID

# Подозрительные сетевые соединения
ss -tupn | grep -v "127.0.0.1\|::1\|10\.\|192\.168\." # исходящие вовне
netstat -an 2>/dev/null | grep ESTABLISHED | awk '{print $5}' | sort | uniq -c

# Анализ auth.log / secure
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head  # brute force IPs
grep "Accepted password\|Accepted publickey" /var/log/auth.log | tail -20  # Debian
grep "Accepted" /var/log/secure | tail -20                                  # RHEL
grep "sudo" /var/log/auth.log | grep -v "^#" | tail -30   # sudo активность

# Изменённые файлы за последнее время
find /bin /sbin /usr/bin /usr/sbin -newer /etc/passwd -type f 2>/dev/null
find / -mtime -1 -not -path "/proc/*" -not -path "/sys/*" -not -path "/run/*" 2>/dev/null | head -30

# SUID/SGID файлы — сравни с бэкапом
find / -perm /4000 -type f 2>/dev/null | sort > /tmp/suid_current.txt
# diff /tmp/suid_baseline.txt /tmp/suid_current.txt

# Проверка целостности пакетов
🔐 rpm -Va 2>/dev/null | grep -v "^\.\.\.\.\.\.\.\.\.\ c\ "  # RHEL — изменённые файлы
🔐 debsums -c 2>/dev/null | head -30                   # Debian — изменённые файлы

# crontabs всех пользователей
🔐 for user in $(cut -d: -f1 /etc/passwd); do
  cron=$(crontab -l -u "$user" 2>/dev/null)
  [ -n "$cron" ] && echo "=== $user ===" && echo "$cron"
done

# Руткит-признаки
# Скрытые директории
ls -la / | grep "^\."                                  # hidden в /
find / -name ".*" -type d -maxdepth 3 2>/dev/null | grep -v "^/proc\|^/sys\|^/home\|^/root\|^/etc"

# auditd — системный аудит
🔐 auditctl -l                                         # активные правила
🔐 ausearch -m execve -ts recent | head -30            # последние запуски программ
🔐 ausearch -f /etc/passwd -ts today                   # доступ к /etc/passwd сегодня
aureport --summary -ts today                           # сводка за сегодня
```

#### Решение

```bash
# Немедленная изоляция (если подтверждена компрометация)
🔐 iptables -I INPUT -j DROP                           # 💣 блокировать весь вход (кроме текущей сессии!)
🔐 iptables -I OUTPUT -j DROP                          # 💣 блокировать весь выход
# ВАЖНО: делай только если ты уже в системе и уверен в своих действиях

# Снятие снэпшота для forensics (до изменений)
🔐 dd if=/dev/sda of=/network-share/forensics.img bs=4M  # 💣 только если есть место

# Базовые правила auditd для мониторинга
🔐 auditctl -w /etc/passwd -p wa -k passwd_changes
🔐 auditctl -w /etc/sudoers -p wa -k sudoers_changes
🔐 auditctl -a always,exit -F arch=b64 -S execve -k exec_log
```

#### Профилактика

```bash
# aide — проверка целостности файлов
🔐 aide --init                                         # создать базу (делай после чистой установки)
🔐 aide --check                                        # сравнить с базой
# Запускай aide --check в cron ежедневно и отправляй diff на почту

# fail2ban для защиты SSH
systemctl status fail2ban
fail2ban-client status sshd                            # список забаненных IP
fail2ban-client set sshd unbanip 1.2.3.4               # разбанить IP
```

---

### 15.9 Аварийное восстановление

#### Симптом
Система не загружается, повреждённая ФС, утеря пароля root, повреждён GRUB/MBR.

#### Диагностика

```bash
# 🔍 Определить тип проблемы
# 1. POST проходит, но не загружается ОС → GRUB проблема
# 2. GRUB загружается, но kernel panic → проблема с ФС или initramfs
# 3. Загружается, но сервисы не стартуют → emergency mode, смотри journalctl -b

# Из GRUB recovery:
# e → редактировать boot entry
# Добавить rd.break — остановить до монтирования root ФС
# В initramfs shell: mount -o rw,remount /sysroot; chroot /sysroot
```

```bash
# Восстановление GRUB (с live-образа)
🔐 # 1. Загрузиться с live-образа
🔐 lsblk                                               # найти системный диск
🔐 mount /dev/sda2 /mnt                                # корневая ФС (обычно sda2 или sda1)
🔐 mount /dev/sda1 /mnt/boot/efi                       # EFI партиция (UEFI)
🔐 mount --bind /dev /mnt/dev
🔐 mount --bind /proc /mnt/proc
🔐 mount --bind /sys /mnt/sys
🔐 chroot /mnt
🔐 grub-install /dev/sda                               # BIOS/MBR
🔐 update-grub                                         # Debian
🔐 grub2-mkconfig -o /boot/grub2/grub.cfg              # RHEL
# UEFI:
🔐 grub-install --target=x86_64-efi --efi-directory=/boot/efi

# Восстановление повреждённой ФС
🔐 umount /dev/sda1                                    # обязательно unmount перед fsck!
🔐 e2fsck -y /dev/sda1                                 # 💣 автоматическое исправление ext4
🔐 xfs_repair -L /dev/sda1                             # 💣 XFS (потеря журнала)
🔐 xfs_repair /dev/sda1                                # XFS без -L (лучше)

# Если ФС в read-only из-за ошибок
🔐 mount -o remount,rw /
🔐 dmesg | grep "EXT4-fs error\|xfs.*error" | tail -10  # конкретные ошибки

# Восстановление MBR (BIOS системы)
🔐 dd if=/dev/zero of=/dev/sda bs=446 count=1          # 💣 только MBR без таблицы разделов
🔐 grub-install /dev/sda                               # восстановить GRUB

# Восстановление пароля root (без live-образа через GRUB)
# 1. GRUB → e → в строке linux добавить: rw init=/bin/bash
# 2. Ctrl+X для загрузки
# 3. В shell:
passwd root
exec /sbin/init
# или
mount -o remount,rw /
passwd
/sbin/reboot -f
```

#### Профилактика

```bash
# Регулярно:
🔐 grub-mkconfig -o /boot/grub/grub.cfg                # актуализировать grub конфиг
# Иметь live-образ на USB готовый к использованию
# Тестировать восстановление в виртуальной среде
# Документировать схему разделов: lsblk -f > /backup/disk_layout.txt
```

---

## Быстрый справочник

### 30 критичных команд для troubleshooting

| Команда | Сценарий применения |
|---------|---------------------|
| `uptime` | Первый взгляд: load average и аптайм |
| `dmesg -T \| tail -30` | Свежие ошибки ядра, OOM, hardware |
| `journalctl -p err --since "10m ago"` | Системные ошибки за последние 10 минут |
| `systemctl list-units --state=failed` | Какие сервисы упали |
| `df -h && df -i` | Полный диск или кончились inode |
| `free -h` | Состояние памяти и swap |
| `vmstat 1 5` | CPU (us/sy/wa/st), swap I/O, runqueue |
| `iostat -xz 1 5` | Диск I/O: утилизация, задержка, очередь |
| `ps aux --sort=-%cpu \| head -10` | Кто жрёт CPU прямо сейчас |
| `ps aux --sort=-%mem \| head -10` | Кто жрёт память прямо сейчас |
| `ss -s` | Статистика TCP/UDP соединений |
| `ss -tulnp` | Кто слушает на каких портах |
| `ss -tan \| awk '{print $1}' \| sort \| uniq -c` | Распределение TCP состояний |
| `lsof +L1` | Удалённые но открытые файлы (скрытый диск) |
| `lsof -p PID` | Все ресурсы конкретного процесса |
| `strace -p PID -c -e trace=all sleep 10` | Что делает зависший процесс |
| `strace -p PID -T 2>&1 \| head -30` | Системные вызовы с временем |
| `tcpdump -i eth0 -n port 80 -c 100` | Анализ HTTP трафика |
| `tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-rst != 0'` | RST пакеты — разрывы соединений |
| `last \| head -20` | История входов пользователей |
| `grep "Failed password" /var/log/auth.log \| tail -20` | Попытки brute force |
| `find / -perm /4000 -type f 2>/dev/null` | SUID файлы — аудит безопасности |
| `dmesg -T \| grep -i oom` | Когда OOM Killer убивал процессы |
| `journalctl -u SERVICE -f` | Логи сервиса в реальном времени |
| `systemd-analyze blame \| head -20` | Что замедляет загрузку системы |
| `iotop -oP` | Какие процессы создают I/O прямо сейчас |
| `mtr --report 8.8.8.8` | Потери пакетов по хопам до хоста |
| `netstat -an 2>/dev/null \| grep :80 \| wc -l` | Количество соединений к порту |
| `cat /proc/PID/limits` | Лимиты конкретного процесса |
| `coredumpctl list` | Core dumps для post-mortem анализа |

---

## Дополнительные материалы

### Документация и книги

- **Brendan Gregg — "Systems Performance"** — библия производительности Linux, автор USE/RED методов, flamegraph
- **Brendan Gregg — "BPF Performance Tools"** — продвинутый профайлинг через eBPF
- **"The Linux Command Line" by William Shotts** — основы, доступна бесплатно на [linuxcommand.org](https://linuxcommand.org)
- **Red Hat Enterprise Linux Documentation** — [access.redhat.com/documentation](https://access.redhat.com/documentation)
- **Ubuntu Server Guide** — [ubuntu.com/server/docs](https://ubuntu.com/server/docs)
- **Arch Wiki** — [wiki.archlinux.org](https://wiki.archlinux.org) — подробные статьи применимы к большинству дистрибутивов

### Онлайн-ресурсы

- **brendangregg.com/linuxperf.html** — Linux Performance Map (постер всех инструментов)
- **explainshell.com** — объяснение любой shell-команды
- **crontab.guru** — валидатор cron-выражений
- **regex101.com** — отладка регулярных выражений с объяснением
- **man7.org** — онлайн man-страницы Linux
- **tldr.sh** — упрощённые man-страницы для быстрого старта

### Инструменты мониторинга

- **htop / btop** — улучшенный top с интерактивным интерфейсом
- **atop** — историческая запись всех метрик (CPU, память, диск, сеть)
- **glances** — обзорный мониторинг всей системы в одном экране
- **netdata** — реальное время метрик с веб-интерфейсом
- **Prometheus + node_exporter** — сбор и хранение метрик для production

### Полезные скрипты и алиасы

```bash
# ~/.bashrc полезные алиасы для troubleshooting
alias myps='ps aux --sort=-%cpu | head -15'
alias mymem='free -h && cat /proc/meminfo | grep -E "^MemAvailable|^Cached|^SwapUsed"'
alias mynet='ss -tulnp'
alias mydisk='df -h && df -i'
alias mylog='journalctl -p err --since "1 hour ago" --no-pager'
alias myfails='systemctl list-units --state=failed'
alias mytop='ps aux --sort=-%cpu | head -5 && echo "---" && ps aux --sort=-%mem | head -5'

# Функция: полный snapshot состояния системы
syssnap() {
  local ts=$(date +%Y%m%d_%H%M%S)
  local out="/tmp/syssnap_${ts}.txt"
  {
    echo "=== SNAPSHOT $ts ==="
    echo "--- uptime ---"; uptime
    echo "--- free ---"; free -h
    echo "--- df ---"; df -h
    echo "--- vmstat ---"; vmstat 1 3
    echo "--- top CPU ---"; ps aux --sort=-%cpu | head -10
    echo "--- top MEM ---"; ps aux --sort=-%mem | head -10
    echo "--- ss ---"; ss -s
    echo "--- failed units ---"; systemctl list-units --state=failed
    echo "--- dmesg ---"; dmesg -T | tail -20
  } | tee "$out"
  echo "Snapshot saved: $out"
}
```

### Checklists для инцидентов

**Checklist первых 5 минут на упавшем сервере:**
1. `uptime` — аптайм и load average
2. `df -h && df -i` — нет ли полного диска/inode
3. `free -h` — достаточно ли памяти
4. `systemctl list-units --state=failed` — упавшие сервисы
5. `dmesg -T | tail -20` — ошибки ядра
6. `journalctl -p err --since "10m ago"` — системные ошибки
7. `ps aux --sort=-%cpu | head -10` — CPU нагрузка
8. `ss -s` — сетевые соединения
9. `last | head -10` — не было ли посторонних входов
10. `journalctl -u <service> -n 50` — логи проблемного сервиса

---

## 1. Освободить место если процесс держит файл

### Диагностика
```bash
# Найти процессы которые держат удалённые файлы
lsof | grep deleted

# Пример вывода:
# nginx  1234  www  10u  REG  8,1  2147483648  /var/log/nginx/access.log (deleted)
# PID = 1234, дескриптор = 10u (число = 10)
```

### Проблема
Файл помечен `deleted` — но место **не освобождается** пока процесс держит файловый дескриптор.  
`rm` не поможет. Нужно усечь файл через `/proc`.

### Решение — усечение через /proc (без убийства процесса)
```bash
# Вариант 1 — через редирект
> /proc/1234/fd/10

# Вариант 2 — через truncate
truncate -s 0 /proc/1234/fd/10

# 1234 = PID процесса
# 10   = номер дескриптора из lsof (колонка FD, без буквы)
```

### Когда убить процесс всё же нужно
```bash
kill -USR1 $(cat /var/run/nginx.pid)   # nginx — мягкая перезагрузка (переоткрывает логи)
kill -HUP <PID>                        # большинство демонов — reload конфига
kill -9 <PID>                          # крайний случай — SIGKILL, не даёт cleanup
```

### Алгоритм при "No space left on device"
```
1. df -h                          → найти забитый раздел
2. lsof | grep deleted            → найти удалённые но открытые файлы
3. truncate -s 0 /proc/PID/fd/N   → освободить место без рестарта
4. du -sh /* | sort -rh | head    → найти крупные файлы если deleted не помогло
5. Настроить logrotate            → чтобы не повторялось
```

---

## 2. Аннотация параметров `top`

### Шапка

```
top - 14:23:11 up 47 days, 3:12,  2 users,  load average: 7.82, 6.91, 5.43
```

| Поле | Что означает |
|------|-------------|
| `up 47 days` | Время работы без перезагрузки |
| `2 users` | Количество залогиненных сессий |
| `load average: 7.82, 6.91, 5.43` | Средняя нагрузка за 1 мин / 5 мин / 15 мин |

**Load average — как читать:**
- Норма = значение ≤ количество ядер (`nproc`)
- `load / cores` — если > 1.0 система перегружена
- Тренд важнее числа: растёт (7→8→9) — проблема нарастает, падает (9→7→5) — пик прошёл

---

```
Tasks: 203 total,  2 running,  201 sleeping,  0 stopped,  0 zombie
```

| Поле | Что означает |
|------|-------------|
| `running` | Процессы активно используют CPU прямо сейчас |
| `sleeping` | Ждут события (I/O, таймер) — норма |
| `stopped` | Приостановлены сигналом (Ctrl+Z) |
| `zombie` | Завершились но родитель не забрал exit code. > 0 — повод искать утечку |

---

```
%Cpu(s): 12.3 us,  2.1 sy,  0.0 ni,  84.2 id,  1.4 wa,  0.0 hi,  0.0 si
```

| Поле | Что означает | Норма |
|------|-------------|-------|
| `us` | User space — CPU на код приложений | Зависит от нагрузки |
| `sy` | System — CPU на ядро ОС | < 10% |
| `ni` | Nice — CPU на процессы с изменённым приоритетом | Обычно 0 |
| `id` | Idle — CPU простаивает | Чем больше тем лучше |
| `wa` | **I/O wait** — CPU ждёт диска | < 5%, > 20% = проблема с диском |
| `hi` | Hardware interrupts | Обычно близко к 0 |
| `si` | Software interrupts | Обычно близко к 0 |

> ⚠️ Высокий `wa` + переполненный swap = система свопирует → всё тормозит

---

```
MiB Mem:  8192.0 total,   412.0 free,  7340.0 used,   440.0 buff/cache
MiB Swap: 2048.0 total,   128.0 free,  1920.0 used,   852.0 avail Mem
```

| Поле | Что означает |
|------|-------------|
| `total` | Физическая RAM |
| `free` | Свободна прямо сейчас |
| `used` | Занята процессами |
| `buff/cache` | Используется ядром для кэша диска — освободится при необходимости |
| `avail Mem` | Реально доступно (free + часть buff/cache) — **смотри сюда, не на free** |
| `Swap used` | Высокий swap = RAM не хватает, система деградирует |

---

### Колонки процессов

```
PID    USER    PR  NI  VIRT    RES    SHR  S  %CPU  %MEM   TIME+    COMMAND
1821   app     20   0  4.2g   6.8g   12m  S  89.3  85.0   142:33   java
```

| Колонка | Что означает |
|---------|-------------|
| `PID` | ID процесса |
| `USER` | Владелец процесса |
| `PR` | Приоритет планировщика |
| `NI` | Nice value (-20 высший приоритет, 19 низший) |
| `VIRT` | Виртуальная память — зарезервировано (включая неиспользуемое) |
| `RES` | **Реальная RAM** — реально занято прямо сейчас |
| `SHR` | Shared memory — разделяется с другими процессами |
| `S` | Статус: `S`=sleep, `R`=running, `Z`=zombie, `D`=I/O wait |
| `%CPU` | Процент CPU |
| `%MEM` | Процент от общей RAM (`RES / total`) |
| `TIME+` | Суммарное CPU-время с момента запуска |
| `COMMAND` | Имя процесса |

> ⚠️ `VIRT >> RES` — нормально (резервирование). `RES` близко к `total` — проблема.

---

### Горячие клавиши в top

| Клавиша | Действие |
|---------|---------|
| `M` | Сортировать по памяти |
| `P` | Сортировать по CPU |
| `k` | Убить процесс (введи PID) |
| `1` | Показать каждое ядро отдельно |
| `q` | Выйти |

---


*Шпаргалка создана для работы в production-средах без GUI. Все команды проверены на Debian 12 / Ubuntu 22.04 и Rocky Linux 9 / CentOS Stream 9. При обнаружении неточностей — создавай issue в репозитории.*
