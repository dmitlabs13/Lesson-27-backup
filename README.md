# Lesson-27-backup

## Задание 
Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.

Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:

директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
имя бекапа должно содержать информацию о времени снятия бекапа;
глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех.
Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.

Запустите стенд на 30 минут.

Убедитесь что резервные копии снимаются.

Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.


## Выполнение

Имеем два сервера 
1. Клиент alp-27-client 192.168.50.213 
2. сервер alp-27-backup 192.168.50.212 


На alp-27-backup cоздана папка и примонтированна к отдельному диску
```
sadmin@alp-27-backup:~$ lsblk
NAME                      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0  30G  0 disk 
├─sda1                      8:1    0   1M  0 part 
├─sda2                      8:2    0   2G  0 part /boot
└─sda3                      8:3    0  28G  0 part 
  └─ubuntu--vg-ubuntu--lv 252:0    0  14G  0 lvm  /
sdb                         8:16   0   2G  0 disk /var/backup
```

создан пользователь и выданы права
```
chown borg:borg /var/backup/
sadmin@alp-27-backup:~$ ls -la /var/backup
total 24
drwxr-xr-x  3 borg borg  4096 Aug 31 18:50 .
drwxr-xr-x 14 root root  4096 Aug 31 18:43 ..
drwx------  2 root root 16384 Aug 31 18:50 lost+found
```
на клиенте создали файл ключа
```
sadmin@alp-27-client:~$ ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/sadmin/.ssh/id_ed25519): key-alp-27
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in key-alp-27
Your public key has been saved in key-alp-27.pub
The key fingerprint is:
SHA256:Ia5X1NyHvsRyfnQ10BoTOHRxFlPfTTazHDwJ4NA+ebI sadmin@alp-27-client
The key's randomart image is:
+--[ED25519 256]--+
|         .ooo**OB|
|         oo=.+=BX|
|      . o +.= =+*|
|     . o . B +  o|
|      . S . X . .|
|     . .   E o . |
|    . .     o .  |
|     .       .   |
|                 |
+----[SHA256]-----+
```


Подключаем папку
```
sadmin@alp-27-client:~$ borg init --encryption=repokey borg@192.168.50.212:/var/backup/repo
borg@192.168.50.212's password: 
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "123wer"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.50.212/var/backup/repo

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
```

```
sadmin@alp-27-client:~$ borg create --stats --list borg@192.168.50.212:/var/backup/repo::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
sadmin@alp-27-client:~$ borg list borg@192.168.50.212:/var/backup/repo
borg@192.168.50.212's password: 
Enter passphrase for key ssh://borg@192.168.50.212/var/backup/repo: 
etc-2026-09-03_19:02:30              Thu, 2026-09-03 19:02:52 [a66a8e08115b2cef6fff775e4c0297a32feb78ea8f7502f1883ac3e4d6ef59d1]
```

смотрим какой есть архив и что в нем
```
sadmin@alp-27-client:~$ borg list borg@192.168.50.212:/var/backup/repo
borg@192.168.50.212's password: 
Enter passphrase for key ssh://borg@192.168.50.212/var/backup/repo: 
etc-2026-09-03_19:02:30              Thu, 2026-09-03 19:02:52 [a66a8e08115b2cef6fff775e4c0297a32feb78ea8f7502f1883ac3e4d6ef59d1]
sadmin@alp-27-client:~$ borg list borg@192.168.50.212:/var/backup/repo::etc-2026-09-03_19:02:30
borg@192.168.50.212's password: 
Enter passphrase for key ssh://borg@192.168.50.212/var/backup/repo: 
drwxr-xr-x root   root          0 Wed, 2026-09-02 06:52:05 etc
lrwxrwxrwx root   root         27 Tue, 2024-04-23 09:40:29 etc/localtime -> /usr/share/zoneinfo/Etc/UTC
lrwxrwxrwx root   root         19 Tue, 2024-04-23 09:40:20 etc/mtab -> ../proc/self/mounts
lrwxrwxrwx root   root         21 Mon, 2024-04-22 13:08:03 etc/os-release -> ../usr/lib/os-release
lrwxrwxrwx root   root         39 Tue, 2024-04-23 09:40:36 etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
lrwxrwxrwx root   root         16 Tue, 2024-04-23 09:40:20 etc/vconsole.conf -> default/keyboard
lrwxrwxrwx root   root         23 Mon, 2024-02-26 12:58:31 etc/vtrgb -> /etc/alternatives/vtrgb
drwxr-xr-x root   root          0 Tue, 2024-04-23 09:45:14 etc/ModemManager
drwxr-xr-x root   root          0 Tue, 2024-04-23 09:45:14 etc/ModemManager/connection.d
drwxr-xr-x root   root          0 Tue, 2024-04-23 09:45:14 etc/ModemManager/fcc-unlock.d
drwxr-xr-x root   root          0 Tue, 2026-09-01 06:55:05 etc/PackageKit
-rw-r--r-- root   root        706 Wed, 2023-11-08 20:35:41 etc/PackageKit/PackageKit.conf
-rw-r--r-- root   root       1718 Sun, 2024-03-31 08:13:09 etc/PackageKit/Vendor.conf
drwxr-xr-x root   root          0 Tue, 2024-04-23 09:40:22 etc/X11
```
восстанавливаем
```
sadmin@alp-27-client:~$ borg extract borg@192.168.50.212:/var/backup/repo::etc-2026-09-03_19:02:30 etc/hostname
borg@192.168.50.212's password: 
Enter passphrase for key ssh://borg@192.168.50.212/var/backup/repo: 
sadmin@alp-27-client:~$
```
настраиваем автоматику/
/etc/systemd/system/borg-backup.service

```
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=123wer"
# Репозиторий
Environment=REPO=borg@192.168.50.212:/var/backup/repo
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}



```
/etc/systemd/system/borg-backup.timer

```
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target

```

Включаем и запускаем службу таймера
```
sadmin@alp-27-client:~$ sudo systemctl enable borg-backup.timer
Created symlink /etc/systemd/system/timers.target.wants/borg-backup.timer → /etc/systemd/system/borg-backup.timer.
sadmin@alp-27-client:~$ sudo systemctl start borg-backup.timer
sadmin@alp-27-client:~$ sudo systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
     Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; preset: enabled)
     Active: active (elapsed) since Thu 2026-09-03 19:38:41 UTC; 8s ago
    Trigger: n/a
   Triggers: ● borg-backup.service

Sep 03 19:38:41 alp-27-client systemd[1]: Started borg-backup.timer - Borg Backup.
```





