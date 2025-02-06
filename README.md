# Домашнее задание Резервное копирование

## Цель домашнего задания  
Научиться настраивать резервное копирование с помощью утилиты Borg.

### Описание домашнего задания  

1.	Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client. (Студент самостоятельно настраивает Vagrant)
2.	Настроить удаленный бэкап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:
  a.	директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB; (Студент самостоятельно настраивает)
  b.	репозиторий для резервных копий должен быть зашифрован ключом или паролем - на усмотрение студента;
  c.	имя бэкапа должно содержать информацию о времени снятия бекапа;
  d.	глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
  e.	резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
  f.	написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на усмотрение студента;
  g.	настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов.


Формат сдачи ДЗ - vagrant + ansible
   
# Выполнение:  


## 1. Создал Vagranfile

```vagrantfile

# -- mode: ruby --
# vi: set ft=ruby :
Vagrant.configure(2) do |config|
  config.vm.box = "bento/ubuntu-22.04"
  #config.vm.provision "ansible" do |ansible|
  #  ansible.playbook = "site.yaml"
  #  ansible.become = "true"
  #end
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 1
  end
  config.vm.define "backup" do |backup|
    backup.vm.network "private_network", ip: "192.168.57.66"
    backup.vm.hostname = "backup"
   # backup.vm.synced_folder "./data", "/home/vagrant/data"
    backup.vm.provider "virtualbox" do |vb|
      unless File.exist?('./storage/backup.vdi')
            vb.customize ['createhd', '--filename', './storage/backup.vdi', '--variant', 'Fixed', '--size', 2176]
            needsController = true
      end
      #vb.customize ["storagectl", :id, "--name", "SATA Controller", "--add", "sata"]
      vb.customize ['storageattach', :id, '--storagectl', 'SATA Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', './storage/backup.vdi']
    end
  end
  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.57.67"
    client.vm.hostname = "client"
   # client.vm.synced_folder "./data", "/home/vagrant/data"
  end
end


```   

## 2. Написал Playbook backup.yml   

```yml
---
- name: set timezone Moscow
  hosts: all
  become: true
  pre_tasks:
    - name: set timezone
      timezone:
        name: Europe/Moscow
  

- name: play config backup
  hosts: backup
  become: true
  roles:
    - install_borgbackup
    - config_backup

  
- name: play config client
  hosts: client
  become: true
  roles:
    - install_borgbackup
    - config_client


- name: copy to backup ssh key
  hosts: backup
  become: true
  roles:
    - add_ssh
  
- name: run backup
  hosts: client
  become: true
  roles:
    - run_backup
    
- name: check borgbackup
  hosts: client
  become: true
  post_tasks:
    - name: show logs jounalctl
      debug:
        var: result.stdout_lines


```   

 


## 3. Далее выполнил команду vagrant up   

```shell
root@debian:/home/guared/backup# vagrant up
Bringing machine 'backup' up with 'virtualbox' provider...
Bringing machine 'client' up with 'virtualbox' provider...

```   

перед запуском playbook'а необходимо установить sshpass   
root@debian:/home/guared/backup/ansible# apt-get update   
root@debian:/home/guared/backup/ansible# apt-get install -y sshpass   

далее запустил playbook   
root@debian:/home/guared/backup/ansible# ansible-playbook -i inventory backup.yml   

```shell
PLAY [set timezone Moscow] ******************************************************************************************************************************************************************

TASK [Gathering Facts] **********************************************************************************************************************************************************************
ok: [192.168.57.67]
ok: [192.168.57.66]

...

PLAY RECAP **********************************************************************************************************************************************************************************
192.168.57.66              : ok=13   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.57.67              : ok=16   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```   
Playbook отработал без ошибок. Также последней задачей выводится логи jounalctl   

  

## 4. Проверяю создание бэкапа   

```shell

root@client:~# borg list borg@192.168.57.66:/var/backup/
Remote: Warning: Permanently added '192.168.57.66' (ED25519) to the list of known hosts.
etc-2025-02-06_12:50:46              Thu, 2025-02-06 12:51:07 [84efe09845d4b55b3d1c27907e7984e69ce76cdcb6dcc54c7ecaf8093f088db4]
etc-2025-02-06_14:19:02              Thu, 2025-02-06 14:19:07 [c2b462bfdfebf9b98e03f30e366c42a13426b741b859a2a6f53ed9580a5e9716]

```   
### 4.1 Посмотрел список фойлов   

```shell
root@client:~# borg list borg@192.168.57.66:/var/backup/::etc-2025-02-06_12:50:46
Remote: Warning: Permanently added '192.168.57.66' (ED25519) to the list of known hosts.
drwxr-xr-x root   root          0 Thu, 2025-02-06 12:48:43 etc
lrwxrwxrwx root   root         19 Fri, 2024-02-16 21:44:26 etc/mtab -> ../proc/self/mounts
lrwxrwxrwx root   root         21 Wed, 2024-02-14 17:47:50 etc/os-release -> ../usr/lib/os-release
lrwxrwxrwx root   root         39 Fri, 2024-02-16 21:44:25 etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
lrwxrwxrwx root   root         13 Tue, 2023-12-05 08:15:51 etc/rmt -> /usr/sbin/rmt
lrwxrwxrwx root   root         23 Fri, 2024-02-16 21:46:20 etc/vtrgb -> /etc/alternatives/vtrgb
drwxr-xr-x root   root          0 Fri, 2024-02-16 21:51:28 etc/ModemManager
drwxr-xr-x root   root          0 Wed, 2023-12-20 08:35:16 etc/ModemManager/connection.d
drwxr-xr-x root   root          0 Wed, 2023-12-20 08:35:16 etc/ModemManager/fcc-unlock.d
drwxr-xr-x root   root          0 Fri, 2024-02-16 21:50:11 etc/PackageKit
-rw-r--r-- root   root        706 Thu, 2022-02-17 16:13:28 etc/PackageKit/PackageKit.conf
-rw-r--r-- root   root       1718 Mon, 2022-03-14 22:11:19 etc/PackageKit/Vendor.conf
...

```   

### 4.2 Пробую достать файл из бекапа   

```shell
root@client:/etc# mv xattr.conf xattr.conf.back
root@client:/etc# ll
...
drwxr-xr-x  4 root root       4096 Feb 16  2024 X11/
-rw-r--r--  1 root root        681 Mar 23  2022 xattr.conf.back
drwxr-xr-x  4 root root       4096 Feb 16  2024 xdg/

root@client:/etc# borg extract borg@192.168.57.66:/var/backup/::etc-2025-02-06_12:50:46 etc/xattr.conf
Remote: Warning: Permanently added '192.168.57.66' (ED25519) to the list of known hosts.

root@client:/etc# ls -l etc/xattr.conf
-rw-r--r-- 1 root root 681 Mar 23  2022 etc/xattr.conf
```


Проверил работу таймера   

```shell
root@client:/# systemctl list-timers --all
NEXT                        LEFT          LAST                        PASSED               UNIT                         ACTIVATES
Thu 2025-02-06 16:43:33 MSK 4min 6s left  Thu 2025-02-06 16:38:33 MSK 53s ago              borg-backup.timer            borg-backup.service
Thu 2025-02-06 17:53:44 MSK 1h 14min left Tue 2024-07-23 21:02:42 MSK 6 months 15 days ago man-db.timer                 man-db.service
Thu 2025-02-06 19:58:25 MSK 3h 18min left Tue 2024-07-23 21:02:42 MSK 6 months 15 days ago fwupd-refresh.timer          fwupd-refresh.service
Thu 2025-02-06 20:34:30 MSK 3h 55min left Tue 2024-07-23 21:02:42 MSK 6 months 15 days ago motd-news.timer              motd-news.service
Fri 2025-02-07 00:00:00 MSK 7h left       n/a                         n/a                  dpkg-db-backup.timer         dpkg-db-backup.service
Fri 2025-02-07 00:00:00 MSK 7h left       Thu 2025-02-06 12:11:26 MSK 4h 27min ago         logrotate.timer              logrotate.service
Fri 2025-02-07 12:24:19 MSK 19h left      Thu 2025-02-06 12:24:19 MSK 4h 15min ago         systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
Sun 2025-02-09 03:10:18 MSK 2 days left   Thu 2025-02-06 12:11:26 MSK 4h 28min ago         e2scrub_all.timer            e2scrub_all.service
Mon 2025-02-10 00:27:36 MSK 3 days left   Thu 2025-02-06 12:22:49 MSK 4h 16min ago         fstrim.timer                 fstrim.service
n/a                         n/a           n/a                         n/a                  apport-autoreport.timer      apport-autoreport.service
n/a                         n/a           n/a                         n/a                  snapd.snap-repair.timer      snapd.snap-repair.service
n/a                         n/a           n/a                         n/a                  ua-timer.timer               ua-timer.service

12 timers listed.
```   


__________________   

end
