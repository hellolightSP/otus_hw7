**Работа с загрузчиком**

**Задание**

- Попасть в систему без пароля несколькими способами.
- Установить систему с LVM, после чего переименовать VG.
- Добавить модуль в initrd.

**Выполнение:**

**Попасть в систему без пароля несколькими способами.**

ДЗ выполняю на сервере с Ubuntu 22.04

- Способ c init=/bin/sh
```
В конце строки с linux добавляем init=/bin/sh и нажимаем сtrl-x для
загрузки в систему

[root@otuslinux ~]# mount -o remount,rw /
root@otuslinux ~]# mount
/ (rw...*)
```

- Способ с rd.break для Ubuntu не нашёл, выполнил немного по другому:
```
В конце строки с linux добавляем rw init=/bin/sh и нажимаем сtrl-x для
загрузки в систему. Попадаем в уже смонтированную на запись ФС:

root(none):/#passwd
Enter new UNIX password: ********
Retupe new UNIX password: ********
passwd: password updated successfully
# For reboot
root(none):/# exec /sbin/init 6

После перезагрузки логинимся в систему с новым паролем root
```


**Установить систему с LVM, после чего переименовать VG**

Работу выполняю на Centos 7 1804

```
[root@localhost ~]# vgs

VG #PV #LV #SN Attr VSize VFree
centos 1 2 0 wz--n- <7.00g 0

[root@localhost ~]# vgrename centos OtusRoot
Volume group "centos" successfully renamed to "OtusRoot"

Правим конфигурационные файлы:

[root@localhost ~]# sed -i 's/centos/otusroot/g' /boot/grub2/grub.cfg && sed -i 's/centos/otusroot/g' /etc/fstab

Пересоздаем initrd image, чтобы он знал новое название Volume Group*

[root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

...

*** Creating image file done ***

*** Creating initramfs image file '/boot/initramfs-3.10.0-862.el7.x86_64.img' done ***
```

Перезагружаемся и проверяем успешную загрузку с новым именем:
```
[root@localhost ~]# vgs
VG #PV #LV #SN Attr VSize VFree
OtusRoot 1 2 0 wz--n- <7.00g 0
```

**Добавить модуль в initrd**

Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. Для того чтобы добавить свой модуль создаем там папку с именем 01test:
```
[root@localhost ~]# mkdir /usr/lib/dracut/modules.d/01test

В нее поместим два скрипта:

1.module-setup.sh - который устанавливает модуль и вызывает скрипт test.sh

2.test.sh - собственно сам вызываемый скрипт, в нём у нас рисуется пингвинчик.

[root@localhost ~]# ls -a

. .. module-setup.sh test.sh

[root@localhost ~]# module-setup.sh

#!/bin/bash

check() {

return 0
}

depends() {

return 0
}

install() {

inst_hook cleanup 00 "${moddir}/test.sh"
}

[root@localhost ~]# vi test.sh

#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console

cat <<'msgend'

Hello! You are in dracut module!

< I'm dracut module >

\

\

    .--.

   |o_o |

   |:_/ |

  //   \ \

 (|     | )

/'\_   _/`\

\___)=(___/
msgend

sleep 10

echo " contining...."

Пересобираем образ initrd и видим пингвина:

[root@localhost ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

[root@localhost ~]# dracut -f -v

Можно проверить/посмотреть какие модули загружены в образ:

[root@localhost ~]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test

test

Далее отредактируем grub.cfg убрав опции rghb и quiet:

... linux16 /vmlinuz-3.10.0-862.el7.x86_64 root=/dev/mapper/OtusRoot-root ro crashkernel=auto rd.lvm.lv=OtusRoot/root rd.lvm.lv=OtusRoot/swap LANG=eng_US.UTF-8

    initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img
...

После перезагрузки и после паузы на 10 секунд появился пингвин в выводе терминала.
```
![Скриншот с пингвином](https://github.com/hellolightSP/otus_hw7/blob/main/init.xcf)
