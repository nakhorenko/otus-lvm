# otus-lvm

```
# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

На диске /dev/sdb создадим временный том для /

```
# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```
```
# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
```
```
# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```
Далее создаём фс и монтируем

```
# mkfs.xfs /dev/vg_root/lv_root 
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
```
# mount /dev/vg_root/lv_root /mnt
# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  /mnt
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```
Копируем корневой каталог в /mnt с помощью утилиты xfsdump

```
# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 8 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

Перенастроим grub на запуск системы из директории /mnt 
Сначала реплицируем необходимые директории в /mnt и меняем конфиг grub2
```
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
```
```
# chroot /mnt
```
```
# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

обновляем образ initrd и меняем в настройке grub2 корневую директорию
```
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;  s/.img//g"` --force; done
```
```
nano /boot/grub2/grub.cfg
rd.lvm.lv=VolGroup00/LogVol00 - rd.lvm.lv=vg_root/lv_root
```
```
# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  /      ------- наша новая / директория
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

Теперь удалим старый логический том VolGroup00 и вместо него создадим аналогичный, но размером 8Gb
```
# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
```
```
# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```
ну и всё в обратном порядке. Создаём файловую систему и монтируем в /mnt новый логический том с ФС, снимаем дамп с логического тома, восстанавливаем в /mnt и переконфигурируем grub2 
```
# mkfs.xfs /dev/VolGroup00/LogVol00
# mount /dev/VolGroup00/LogVol00 /mnt
# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
# chroot /mnt/
# grub2-mkconfig -o /boot/grub2/grub.cfg
```
```
# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0    8G  0 lvm  /
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

Выделяем логический том под /var

```
# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```
```
# vgcreate vg_var /dev/sdc /dev/sdd
```
```
# lvcreate -n lv_var vg_var -L 950M -m1
```
Создаем xfs файловую систему
```
# mkfs.ext4 /dev/vg_var/lv_var
```
монтируем в /mnt, копируем содержимое /var/* в /mnt
```
# mount /dev/vg_var/lv_var /mnt
# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:0    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt
└─vg_var-lv_var_rimage_0 253:4    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt
└─vg_var-lv_var_rimage_1 253:6    0  952M  0 lvm  
  └─vg_var-lv_var        253:7    0  952M  0 lvm  /mnt
sde                        8:64   0    1G  0 disk
```
Чтобы ничего не зафакапить, делаем копию старой директории /var
```
# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
```
# umount /mnt
```
Монтируем логический том в /var
```
# mount /dev/vg_var/lv_var /var
```
Дальше надо записать точки автоматического монитрования в fstab
```
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
UUID="b6323dc0-59d8-4f16-b37b-8b3860f01741" /var ext4 defaults 0 0        ---- точка монтирования появилась
```
Выделяем том под /home

```
# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
```
```
# lvs
  LV          VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00    VolGroup00 -wi-ao----   8.00g                                                    
  LogVol01    VolGroup00 -wi-ao----   1.50g                                                    
  LogVol_Home VolGroup00 -wi-a-----   2.00g                                                    
  lv_var      vg_var     rwi-aor--- 952.00m                                    100.00
```
```
# mkfs.xfs /dev/VolGroup00/LogVol_Home
```
```
# mount /dev/VolGroup00/LogVol_Home /mnt/
# cp -aR /home/* /mnt/
# rm -rf /home/*
# umount /mnt
# mount /dev/VolGroup00/LogVol_Home /home/
```
```
# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
UUID="b6323dc0-59d8-4f16-b37b-8b3860f01741" /var ext4 defaults 0 0
UUID="d66171ea-de01-4d55-93f3-8e3011c3d027" /home xfs defaults 0 0   ----- точка монтирования создана
```
Делаем для /home том для снэпшотов

```
# touch /home/file{1..20}
# ls -la /home
total 0
drwxr-xr-x.  3 root    root    292 Jan 17 12:49 .
drwxr-xr-x. 18 root    root    239 Jan 17 11:25 ..
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file1
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file10
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file11
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file12
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file13
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file14
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file15
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file16
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file17
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file18
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file19
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file2
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file20
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file3
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file4
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file5
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file6
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file7
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file8
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file9
```
Создаём снэпшот
```
# lvcreate -L 100M -s -n home_snap /dev/VolGroup00/LogVol_Home
```
Удалим часть файлов из папки home
```
# rm -f /home/file{11..20}
```
```
# ls -la /home
total 0
drwxr-xr-x.  3 root    root    152 Jan 17 12:52 .
drwxr-xr-x. 18 root    root    239 Jan 17 11:25 ..
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file1
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file10
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file2
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file3
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file4
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file5
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file6
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file7
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file8
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file9
```
Восстанавливаем содержимое /home из снэпшота
```
# umount /home
#  lvconvert --merge /dev/VolGroup00/home_snap 
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
```
Возвращаем /home 
```
# mount /home
# ls -la /home
total 0
drwxr-xr-x.  3 root    root    292 Jan 17 12:49 .
drwxr-xr-x. 18 root    root    239 Jan 17 11:25 ..
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file1
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file10
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file11
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file12
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file13
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file14
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file15
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file16
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file17
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file18
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file19
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file2
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file20
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file3
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file4
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file5
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file6
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file7
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file8
-rw-r--r--.  1 root    root      0 Jan 17 12:49 file9
```
