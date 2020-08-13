# Работа с Logical Volume Management lvm

В данной работе представлены примеры работы с LVM. Все команды производятся с правами root.  

## Уменьшить том под / (корневой каталог) до 8G

Подготовим временный том для / раздела.

Создаем раздел pv:

`# pvcreate /dev/sdb`

*Physical volume "/dev/sdb" successfully created.*

Создаем раздел vm:

`# vgcreate vg_root /dev/sdb`

*Volume group "vg_root" successfully created*

Создаем раздел lv:

`# lvcreate -n lv_root -l +100%FREE /dev/vg_root`

*Logical volume "lv_root" created.*

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

`# mkfs.xfs /dev/vg_root/lv_root`

`# mount /dev/vg_root/lv_root /mnt`

Этой командой скопируем все данные с / раздела в /mnt:

`xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt`

Переконфигурируем grub для того, чтобы при старте перейти в новый /

Создадим все основные каталоги:

`# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done`

Создадим имитацию текущего root -> сделаем в него chroot и обновим grub:

`# chroot /mnt/`
`# grub2-mkconfig -o /boot/grub2/grub.cfg`

Обновим образ initrd.

``# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
s/.img//g"` --force; done``

В файле */boot/grub2/grub.cfg* нужно заменить предыдущий том lvm на вновь созданный: *rd.lvm.lv=VolGroup00/LogVol00* => *rd.lvm.lv=vg_root/lv_root*

Перезагружаемся успешно с новым рут томом. Убедиться в этом можно посмотрев вывод lsblk:

Изменяем размер старой VG и возвращаем на него рут.
Для этого удаляем старый LV размеров в 40G:

`# lvremove /dev/VolGroup00/LogVol00`

*Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y

Logical volume "LogVol00" successfully removed*

Создаем новый на 8G:

`# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00`

*WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y  Wiping xfs signature on /dev/VolGroup00 LogVol00.  Logical volume "LogVol00" created.*

Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

`# mkfs.xfs /dev/VolGroup00/LogVol00`
`# mount /dev/VolGroup00/LogVol00 /mnt`

Обратно скопируем все данные с / раздела в /mnt:

`# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt`

Так же как в первый раз переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg:

`# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done`

`# chroot /mnt/`

`# grub2-mkconfig -o /boot/grub2/grub.cfg`

``# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done``

После чего можно успешно перезагружаться в новый / и удалять временную Volume Group:

`# lvremove /dev/vg_root/lv_root`

*Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y

Logical volume "lv_root" successfully removed*
`# vgremove /dev/vg_root`

*Volume group "vg_root" successfully removed*

`# pvremove /dev/sdb`

*Labels on physical volume "/dev/sdb" successfully wiped.*

## Выделить lvm том в зеркале под /var

На дисках sdc и sdd создаем lvm том с зеркалом на 750 Мб:

`# pvcreate /dev/sdc /dev/sdd`

*Physical volume "/dev/sdc" successfully created.

Physical volume "/dev/sdd" successfully created.*

`# vgcreate vg_var /dev/sdc /dev/sdd`

*Volume group "vg_var" successfully created*

`# lvcreate -L 750M -m1 -n lv_var vg_var`

*Rounding up size to full physical extent 752.00 MiB

Logical volume "lv_var" created.*

Создаем на нем ФС:

`# mkfs.ext4 /dev/vg_var/lv_var`

 Перемещаем туда /var:

 `# mount /dev/vg_var/lv_var /mnt`
 `# cp -aR /var/* /mnt/`

 На всякий случай сохраняем содержимое старого var:

 `# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar`

 Монтируем новый var в каталог /var:

 `# umount /mnt`

 `# mount /dev/vg_var/lv_var /var`

 Правим fstab для автоматического монтирования /var:

 *echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab*

 Перезагружаемся и командой lsblk проверяем.

## Выделить lvm том gjl /home

`# pvcreate /dev/sdb`

`# vgcreate vg_home /dev/sdb`

`# lvcreate -n lv_home -L 5G /dev/vg_home`

`# mkfs.xfs /dev/vg_home/lv_home`

`# mount /dev/vg_home/lv_home /mnt/`

cp -aR /home/* /mnt/

`# rm -rf /home/*`

`# umount /mnt`

`# mount /dev/vg_home/lv_home /home/`

*echo "`blkid | grep lv_home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab*
