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
