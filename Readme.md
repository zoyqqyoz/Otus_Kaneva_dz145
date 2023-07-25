Vagrant-стенд c PAM

Цель домашнего задания
Научиться создавать пользователей и добавлять им ограничения

Описание домашнего задания
Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

Создадим Vagrantfile, запустим его и подключимся к ВМ:

```
neva@Uneva:~$ vagrant up
Bringing machine 'pam' up with 'virtualbox' provider...
==> pam: Box 'centos/8' could not be found. Attempting to find and install...
    pam: Box Provider: virtualbox
    pam: Box Version: >= 0
==> pam: Loading metadata for box 'centos/8'
    pam: URL: https://vagrantcloud.com/centos/8
==> pam: Adding box 'centos/8' (v2011.0) for provider: virtualbox
    pam: Downloading: https://vagrantcloud.com/centos/boxes/8/versions/2011.0/providers/virtualbox.box
Download redirected to host: cloud.centos.org
    pam: Calculating and comparing box checksum...
==> pam: Successfully added box 'centos/8' (v2011.0) for 'virtualbox'!
==> pam: Importing base box 'centos/8'...
==> pam: Matching MAC address for NAT networking...
==> pam: Checking if box 'centos/8' version '2011.0' is up to date...
==> pam: Setting the name of the VM: neva_pam_1690206532151_95772
==> pam: Clearing any previously set network interfaces...
==> pam: Preparing network interfaces based on configuration...
    pam: Adapter 1: nat
    pam: Adapter 2: hostonly
==> pam: Forwarding ports...
    pam: 22 (guest) => 2222 (host) (adapter 1)
==> pam: Running 'pre-boot' VM customizations...
==> pam: Booting VM...
==> pam: Waiting for machine to boot. This may take a few minutes...
    pam: SSH address: 127.0.0.1:2222
    pam: SSH username: vagrant
    pam: SSH auth method: private key
    pam:
    pam: Vagrant insecure key detected. Vagrant will automatically replace
    pam: this with a newly generated keypair for better security.
    pam:
    pam: Inserting generated public key within guest...
    pam: Removing insecure key from the guest if it's present...
    pam: Key inserted! Disconnecting and reconnecting using new SSH key...
==> pam: Machine booted and ready!
==> pam: Checking for guest additions in VM...
    pam: No guest additions were detected on the base box for this VM! Guest
    pam: additions are required for forwarded ports, shared folders, host only
    pam: networking, and more. If SSH fails on this machine, please install
    pam: the guest additions and repackage the box to continue.
    pam:
    pam: This is not an error message; everything may continue to work properly,
    pam: in which case you may ignore this message.
==> pam: Setting hostname...
==> pam: Configuring and enabling network interfaces...
==> pam: Running provisioner: shell...
    pam: Running: inline script
neva@Uneva:~$ vagrant ssh
```
Создаём пользователя otusadm и otus:

```
sudo useradd otusadm && sudo useradd otus
```

Создаём пользователям пароли:

```
[vagrant@pam ~]$ echo "Otus2022!" | sudo passwd --stdin otusadm && echo "Otus2022!" | sudo passwd --stdin otus
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
Changing password for user otus.
passwd: all authentication tokens updated successfully.
```

Создаём группу admin:

```
sudo groupadd -f admin
```

Добавляем пользователей vagrant,root и otusadm в группу admin:

```
[vagrant@pam ~]$ sudo -i
[root@pam ~]# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
```

После создания пользователей, нужно проверить, что они могут подключаться по SSH к нашей ВМ. Для этого пытаемся подключиться с хостовой машины: 

```
neva@Uneva:~$ ssh otus@192.168.56.10
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:yOGy/n241NDTi9FGF5JORPAtEl3rl6DpqQzRjMIGlvk.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.10' (ED25519) to the list of known hosts.
otus@192.168.56.10's password:
[otus@pam ~]$ whoami
otus
[otus@pam ~]$ exit
logout
Connection to 192.168.56.10 closed.
neva@Uneva:~$ ssh otusadm@192.168.56.10
otusadm@192.168.56.10's password:
[otusadm@pam ~]$ whoami
otusadm
[otusadm@pam ~]$ exit
logout
Connection to 192.168.56.10 closed.
neva@Uneva:~$
```

Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни. Проверим, что пользователи root, vagrant и otusadm есть в группе admin:

```
[root@pam ~]# cat /etc/group | grep admin
printadmin:x:994:
admin:x:1003:otusadm,root,vagrant
```

Выберем метод PAM-аутентификации, так как у нас используется только ограничение по времени, то было бы логично использовать метод pam_time, однако, данный метод не работает с локальными группами пользователей, и, получается, что использование данного метода добавит нам большое количество однообразных строк с разными пользователями. В текущей ситуации лучше написать небольшой скрипт контроля и использовать модуль pam_exec

Создадим файл-скрипт /usr/local/bin/login.sh

```
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```

В скрипте подписаны все условия. Скрипт работает по принципу: 
Если сегодня суббота или воскресенье, то нужно проверить, входит ли пользователь в группу admin, если не входит — то подключение запрещено. При любых других вариантах подключение разрешено. 

Добавим права на исполнение файла:

```
[root@pam ~]# chmod +x /usr/local/bin/login.sh
```

Укажем в файле /etc/pam.d/sshd вызов модуля pam_exec.so и путь к нашему скрипту в секции auth:

```
[root@pam ~]# vi /etc/pam.d/sshd
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
auth       required     pam_exec.so /usr/local/bin/login.sh
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    include      postlogin
```

Меняем дату на субботу:

```
[root@pam ~]# date 082712302022.00
Sat Aug 27 12:30:00 UTC 2022
```

и пробуем подключиться под пользователем otus:

```
neva@Uneva:~$ ssh otus@192.168.56.10
otus@192.168.56.10's password:
Permission denied, please try again.
otus@192.168.56.10's password:
```

Доступ запрещён. Пробуем зайти под otusadm:

```
neva@Uneva:~$ ssh otusadm@192.168.56.10
otusadm@192.168.56.10's password:
Last login: Tue Jul 25 07:53:42 2023 from 192.168.56.1
[otusadm@pam ~]$
```


Доступ открыт.





```


