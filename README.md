# Резервное копирование
##Что нужно сделать?

###Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.
Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup.  
Резервные копии должны соответствовать следующим критериям:  
    - директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;  
    - репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;  
    - имя бекапа должно содержать информацию о времени снятия бекапа;  
    - глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех.  
    - Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;  
    - резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;  
    - написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;  
    - настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.    
    
```
user@lab:~/dz/otus27/ansible$ ansible-playbook provision.yml

PLAY [Config backup server] ************************************************************

TASK [Create user] *********************************************************************
[DEPRECATION WARNING]: Encryption using the Python crypt module is deprecated. The
Python crypt module is deprecated and will be removed from Python 3.13. Install the
passlib library for continued encryption functionality. This feature will be removed in
 version 2.17. Deprecation warnings can be disabled by setting
deprecation_warnings=False in ansible.cfg.
changed: [backup]

TASK [Create .ssh directory] ***********************************************************
ok: [backup]

TASK [Create authorized_keys] **********************************************************
changed: [backup]

TASK [Create a new ext4 partition] *****************************************************
ok: [backup]

TASK [Format the volume with ext4 fs] **************************************************
ok: [backup]

TASK [mount the partition on /var/backup] **********************************************
ok: [backup]

TASK [Target directory backup] *********************************************************
changed: [backup]

PLAY [Install borgbackup] **************************************************************

TASK [Gathering Facts] *****************************************************************
ok: [backup]
ok: [client]

TASK [install borg] ********************************************************************
ok: [client]
ok: [backup]

PLAY [Config client server] ************************************************************

TASK [install sshpass] *****************************************************************
ok: [client]

TASK [copy key sshpass] ****************************************************************
ok: [client]

TASK [Create a 2048-bit SSH key for user root] *****************************************
ok: [client]

TASK [copy key] ************************************************************************
changed: [client]

TASK [copy borg-backup.service] ********************************************************
ok: [client]

TASK [copy borg-backup.timer] **********************************************************
ok: [client]

TASK [reload systemctl] ****************************************************************
changed: [client]

TASK [enable borg-backup.timer] ********************************************************
ok: [client]

TASK [enable borg-backup.service] ******************************************************
changed: [client]

PLAY RECAP *****************************************************************************
backup                     : ok=9    changed=3    unreachable=0    failed=0    skipped=0                                                                                                                            rescued=0    ignored=0
client                     : ok=11   changed=3    unreachable=0    failed=0    skipped=0                                                                                                                            rescued=0    ignored=0
```
Для бэкапов подключен отдельный жеский диск и примонтирован /var/backup
```
vagrant@backup:~$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0                       7:0    0  63.4M  1 loop /snap/core20/1974
loop1                       7:1    0 111.9M  1 loop /snap/lxd/24322
loop2                       7:2    0  53.3M  1 loop /snap/snapd/19457
loop3                       7:3    0  63.7M  1 loop /snap/core20/2434
loop4                       7:4    0  89.4M  1 loop /snap/lxd/31333
sda                         8:0    0   128G  0 disk
├─sda1                      8:1    0     1M  0 part
├─sda2                      8:2    0     2G  0 part /boot
└─sda3                      8:3    0   126G  0 part
  └─ubuntu--vg-ubuntu--lv 253:0    0    63G  0 lvm  /
sdb                         8:16   0     2G  0 disk
└─sdb1                      8:17   0     2G  0 part /var/backup
vagrant@backup:~$
```
На клиенте запущен таймер для выполнения  borg-backup.service 1 раз в 5 минут
```
root@client:~# systemctl list-timers --all
NEXT                        LEFT          LAST                        PASSED              UNIT                         ACTIVATES
Sat 2025-01-11 17:19:29 UTC 4s left       n/a                         n/a                 systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Sat 2025-01-11 17:20:00 UTC 34s left      Sat 2025-01-11 17:10:00 UTC 9min ago            sysstat-collect.timer        sysstat-collect.service
Sat 2025-01-11 17:20:12 UTC 47s left      n/a                         n/a                 borg-backup.timer            borg-backup.service
Sat 2025-01-11 17:45:26 UTC 26min left    Wed 2024-01-10 23:56:08 UTC 1 year 0 months ago fwupd-refresh.timer          fwupd-refresh.service
Sat 2025-01-11 18:13:12 UTC 53min left    Wed 2024-01-10 23:56:08 UTC 1 year 0 months ago fstrim.timer                 fstrim.service
Sat 2025-01-11 22:06:54 UTC 4h 47min left Wed 2024-01-10 23:56:08 UTC 1 year 0 months ago man-db.timer                 man-db.service
Sat 2025-01-11 22:35:32 UTC 5h 16min left Thu 2024-01-11 00:04:58 UTC 1 year 0 months ago plocate-updatedb.timer       plocate-updatedb.service
Sun 2025-01-12 00:00:00 UTC 6h left       n/a                         n/a                 dpkg-db-backup.timer         dpkg-db-backup.service
Sun 2025-01-12 00:00:00 UTC 6h left       Sat 2025-01-11 17:04:48 UTC 14min ago           logrotate.timer              logrotate.service
Sun 2025-01-12 00:07:00 UTC 6h left       n/a                         n/a                 sysstat-summary.timer        sysstat-summary.service
Sun 2025-01-12 03:10:28 UTC 9h left       Sat 2025-01-11 17:05:12 UTC 14min ago           e2scrub_all.timer            e2scrub_all.service
n/a                         n/a           n/a                         n/a                 apport-autoreport.timer      apport-autoreport.service
n/a                         n/a           n/a                         n/a                 snapd.snap-repair.timer      snapd.snap-repair.service
n/a                         n/a           n/a                         n/a                 ua-timer.timer               ua-timer.service

14 timers listed.
```
Лог выпонения
```
root@client:~# sudo journalctl -u borg-backup.service
Jan 11 17:10:28 client systemd[1]: Starting Borg Backup...
Jan 11 17:10:35 client borg[3204]: ------------------------------------------------------------------------------
Jan 11 17:10:35 client borg[3204]: Repository: ssh://borg@192.168.50.10/var/backup
Jan 11 17:10:35 client borg[3204]: Archive name: etc-2025-01-11_17:10:29
Jan 11 17:10:35 client borg[3204]: Archive fingerprint: 22df597bb050fc75cf3810adda86a49a5c1f1b1d0900e744c976b84501d96c0d
Jan 11 17:10:35 client borg[3204]: Time (start): Sat, 2025-01-11 17:10:32
Jan 11 17:10:35 client borg[3204]: Time (end):   Sat, 2025-01-11 17:10:35
Jan 11 17:10:35 client borg[3204]: Duration: 2.48 seconds
Jan 11 17:10:35 client borg[3204]: Number of files: 700
Jan 11 17:10:35 client borg[3204]: Utilization of max. archive size: 0%
Jan 11 17:10:35 client borg[3204]: ------------------------------------------------------------------------------
Jan 11 17:10:35 client borg[3204]:                        Original size      Compressed size    Deduplicated size
Jan 11 17:10:35 client borg[3204]: This archive:                2.11 MB            930.00 kB              7.48 kB
Jan 11 17:10:35 client borg[3204]: All archives:                6.35 MB              2.79 MB              1.07 MB
Jan 11 17:10:35 client borg[3204]:                        Unique chunks         Total chunks
Jan 11 17:10:35 client borg[3204]: Chunk index:                     695                 2081
Jan 11 17:10:35 client borg[3204]: ------------------------------------------------------------------------------
Jan 11 17:10:42 client systemd[1]: borg-backup.service: Deactivated successfully.
Jan 11 17:10:42 client systemd[1]: Finished Borg Backup.
Jan 11 17:10:42 client systemd[1]: borg-backup.service: Consumed 5.940s CPU time.
Jan 11 17:15:12 client systemd[1]: Starting Borg Backup...
Jan 11 17:15:19 client borg[4065]: ------------------------------------------------------------------------------
Jan 11 17:15:19 client borg[4065]: Repository: ssh://borg@192.168.50.10/var/backup
Jan 11 17:15:19 client borg[4065]: Archive name: etc-2025-01-11_17:15:12
Jan 11 17:15:19 client borg[4065]: Archive fingerprint: a4d99db68dc693dc045271e770aac9deb5175f612f4858554426a98aa5ff4fcb
Jan 11 17:15:19 client borg[4065]: Time (start): Sat, 2025-01-11 17:15:16
Jan 11 17:15:19 client borg[4065]: Time (end):   Sat, 2025-01-11 17:15:19
Jan 11 17:15:19 client borg[4065]: Duration: 2.33 seconds
Jan 11 17:15:19 client borg[4065]: Number of files: 705
Jan 11 17:15:19 client borg[4065]: Utilization of max. archive size: 0%
Jan 11 17:15:19 client borg[4065]: ------------------------------------------------------------------------------
Jan 11 17:15:19 client borg[4065]:                        Original size      Compressed size    Deduplicated size
Jan 11 17:15:19 client borg[4065]: This archive:                2.12 MB            931.95 kB            913.30 kB
Jan 11 17:15:19 client borg[4065]: All archives:                2.12 MB            931.30 kB            985.80 kB
Jan 11 17:15:19 client borg[4065]:                        Unique chunks         Total chunks
Jan 11 17:15:19 client borg[4065]: Chunk index:                     677                  693
Jan 11 17:15:19 client borg[4065]: ------------------------------------------------------------------------------
Jan 11 17:15:26 client systemd[1]: borg-backup.service: Deactivated successfully.
Jan 11 17:15:26 client systemd[1]: Finished Borg Backup.
Jan 11 17:15:26 client systemd[1]: borg-backup.service: Consumed 6.080s CPU time.
Jan 11 17:20:18 client systemd[1]: Starting Borg Backup...
Jan 11 17:20:23 client borg[4095]: ------------------------------------------------------------------------------
Jan 11 17:20:23 client borg[4095]: Repository: ssh://borg@192.168.50.10/var/backup
Jan 11 17:20:23 client borg[4095]: Archive name: etc-2025-01-11_17:20:19
Jan 11 17:20:23 client borg[4095]: Archive fingerprint: 2449435ebfda399dbd14debae0a5de3b873bae4c757c6c649a03ef1d3f02b664
Jan 11 17:20:23 client borg[4095]: Time (start): Sat, 2025-01-11 17:20:23
Jan 11 17:20:23 client borg[4095]: Time (end):   Sat, 2025-01-11 17:20:23
Jan 11 17:20:23 client borg[4095]: Duration: 0.45 seconds
Jan 11 17:20:23 client borg[4095]: Number of files: 705
Jan 11 17:20:23 client borg[4095]: Utilization of max. archive size: 0%
Jan 11 17:20:23 client borg[4095]: ------------------------------------------------------------------------------
Jan 11 17:20:23 client borg[4095]:                        Original size      Compressed size    Deduplicated size
Jan 11 17:20:23 client borg[4095]: This archive:                2.12 MB            931.95 kB                645 B
Jan 11 17:20:23 client borg[4095]: All archives:                4.23 MB              1.86 MB            986.44 kB
Jan 11 17:20:23 client borg[4095]:                        Unique chunks         Total chunks
Jan 11 17:20:23 client borg[4095]: Chunk index:                     678                 1386
Jan 11 17:20:23 client borg[4095]: ------------------------------------------------------------------------------
```
Бэкапы в репозитории
```
root@client:~#  borg list borg@192.168.50.10:/var/backup
Enter passphrase for key ssh://borg@192.168.50.10/var/backup:
etc-2025-01-11_17:15:12              Sat, 2025-01-11 17:15:16 [a4d99db68dc693dc045271e770aac9deb5175f612f4858554426a98aa5ff4fcb]
etc-2025-01-11_17:20:19              Sat, 2025-01-11 17:20:23 [2449435ebfda399dbd14debae0a5de3b873bae4c757c6c649a03ef1d3f02b664]
```
Переместим /etc
```
root@client:~# mv  /etc /root/
root@client:~# ls -la /
total 2097228
drwxr-xr-x  19 0 0       4096 Jan 11 17:21 .
drwxr-xr-x  19 0 0       4096 Jan 11 17:21 ..
lrwxrwxrwx   1 0 0          7 Aug 10  2023 bin -> usr/bin
drwxr-xr-x   4 0 0       4096 Jan 11  2024 boot
drwxr-xr-x  19 0 0       3980 Jan 11 17:21 dev
drwxr-xr-x   9 0 0       4096 Jan 11 17:21 etc
drwxr-xr-x   3 0 0       4096 Jan 10  2024 home
lrwxrwxrwx   1 0 0          7 Aug 10  2023 lib -> usr/lib
lrwxrwxrwx   1 0 0          9 Aug 10  2023 lib32 -> usr/lib32
lrwxrwxrwx   1 0 0          9 Aug 10  2023 lib64 -> usr/lib64
lrwxrwxrwx   1 0 0         10 Aug 10  2023 libx32 -> usr/libx32
drwx------   2 0 0      16384 Jan 10  2024 lost+found
drwxr-xr-x   2 0 0       4096 Aug 10  2023 media
drwxr-xr-x   2 0 0       4096 Aug 10  2023 mnt
drwxr-xr-x   2 0 0       4096 Aug 10  2023 opt
dr-xr-xr-x 191 0 0          0 Jan 11 17:04 proc
drwx------   7 0 0       4096 Jan 11 17:21 root
drwxr-xr-x  29 0 0        880 Jan 11 17:12 run
lrwxrwxrwx   1 0 0          8 Aug 10  2023 sbin -> usr/sbin
drwxr-xr-x   6 0 0       4096 Jan 11 17:10 snap
drwxr-xr-x   2 0 0       4096 Aug 10  2023 srv
-rw-------   1 0 0 2147483648 Jan 10  2024 swap.img
dr-xr-xr-x  13 0 0          0 Jan 11 17:04 sys
drwxrwxrwt  14 0 0       4096 Jan 11 17:21 tmp
drwxr-xr-x  14 0 0       4096 Aug 10  2023 usr
drwxr-xr-x  13 0 0       4096 Aug 10  2023 var
root@client:~#
```
Востановим из бэкапа
```
root@client:~# borg extract --list borg@192.168.50.10:/var/backup/::etc-2025-01-11_17:20:19 /etc
Enter passphrase for key ssh://borg@192.168.50.10/var/backup:
etc
etc/localtime
etc/mtab
etc/os-release
etc/resolv.conf
etc/apt
etc/apt/sources.list
etc/apt/apt.conf.d
etc/apt/apt.conf.d/10periodic
etc/apt/apt.conf.d/15update-stamp
etc/apt/apt.conf.d/20apt-esm-hook.conf
etc/apt/apt.conf.d/20archive
etc/apt/apt.conf.d/50command-not-found
etc/apt/apt.conf.d/99update-notifier
etc/apt/apt.conf.d/01-vendor-ubuntu
etc/apt/apt.conf.d/01autoremove
etc/apt/apt.conf.d/20auto-upgrades
etc/apt/apt.conf.d/20packagekit
etc/apt/apt.conf.d/20snapd.conf
etc/apt/apt.conf.d/50unattended-upgrades
etc/apt/apt.conf.d/70debconf
etc/apt/apt.conf.d/99needrestart
etc/apt/auth.conf.d
etc/apt/keyrings
etc/apt/preferences.d
_________________


_________________
etc/apm/other.d
etc/apm/other.d/50ifplugd
etc/apm/resume.d
etc/apm/resume.d/80ifplugd
etc/apm/scripts.d
etc/apm/scripts.d/ifplugd
etc/apm/suspend.d
etc/apm/suspend.d/20ifplugd
etc/ifplugd
etc/ifplugd/action.d
etc/ifplugd/action.d/ifupdown
etc/ifplugd/ifplugd.action
etc/ifplugd/ifplugd.conf
etc/gshadow
etc/vagrant_box_build_time
etc/group

root@client:~# ls -al /
total 2097228
drwxr-xr-x  19 root 0       4096 Jan 11 17:21 .
drwxr-xr-x  19 root 0       4096 Jan 11 17:21 ..
lrwxrwxrwx   1 root 0          7 Aug 10  2023 bin -> usr/bin
drwxr-xr-x   4 root 0       4096 Jan 11  2024 boot
drwxr-xr-x  19 root 0       3980 Jan 11 17:21 dev
drwxr-xr-x   9 root 0       4096 Jan 11 17:22 etc
drwxr-xr-x   3 root 0       4096 Jan 10  2024 home
lrwxrwxrwx   1 root 0          7 Aug 10  2023 lib -> usr/lib
lrwxrwxrwx   1 root 0          9 Aug 10  2023 lib32 -> usr/lib32
lrwxrwxrwx   1 root 0          9 Aug 10  2023 lib64 -> usr/lib64
lrwxrwxrwx   1 root 0         10 Aug 10  2023 libx32 -> usr/libx32
drwx------   2 root 0      16384 Jan 10  2024 lost+found
drwxr-xr-x   2 root 0       4096 Aug 10  2023 media
drwxr-xr-x   2 root 0       4096 Aug 10  2023 mnt
drwxr-xr-x   2 root 0       4096 Aug 10  2023 opt
dr-xr-xr-x 167 root 0          0 Jan 11 17:04 proc
drwx------   7 root 0       4096 Jan 11 17:21 root
drwxr-xr-x  29 root 0        880 Jan 11 17:12 run
lrwxrwxrwx   1 root 0          8 Aug 10  2023 sbin -> usr/sbin
drwxr-xr-x   6 root 0       4096 Jan 11 17:10 snap
drwxr-xr-x   2 root 0       4096 Aug 10  2023 srv
-rw-------   1 root 0 2147483648 Jan 10  2024 swap.img
dr-xr-xr-x  13 root 0          0 Jan 11 17:04 sys
drwxrwxrwt  13 root 0       4096 Jan 11 17:25 tmp
drwxr-xr-x  14 root 0       4096 Aug 10  2023 usr
drwxr-xr-x  13 root 0       4096 Aug 10  2023 var
root@client:~# ls -la
```
