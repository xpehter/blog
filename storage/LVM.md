### Увеличение размера диска
_Рассматривается вариант, когда диск размечен в GPT и используется LVM с файловой системой ext4_

LVM - это прослойка между дисками и ОС. Которая добовляет гибкости в работе с разделами. В общем виде схема такая:

`/dev/sda <-> PV(physical volume) <-> VG(volume group) <-> LV(logical volume) <-> ext4`

Файловая система устанавливается поверх LV. LV состоит из VG. VG состоит из PV. PV - состоит из устройств.

- Для примера исходные данные
   ```
   # df -h
   Filesystem                         Size  Used Avail Use% Mounted on
   udev                               912M     0  912M   0% /dev
   tmpfs                              189M  860K  188M   1% /run
   /dev/mapper/ubuntu--vg-ubuntu--lv  6.4G  3.6G  2.5G  60% /
   tmpfs                              942M     0  942M   0% /dev/shm
   tmpfs                              5.0M     0  5.0M   0% /run/lock
   tmpfs                              942M     0  942M   0% /sys/fs/cgroup
   /dev/loop1                          90M   90M     0 100% /snap/core/6818
   /dev/loop0                          91M   91M     0 100% /snap/core/6350
   /dev/sda2                          976M   77M  833M   9% /boot
   /dev/sda1                          511M  6.1M  505M   2% /boot/efi
   tmpfs                              189M  8.0K  189M   1% /run/user/1000
   
   # lsblk
   NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   loop0                       7:0    0   91M  1 loop /snap/core/6350
   loop1                       7:1    0 89.4M  1 loop /snap/core/6818
   sda                         8:0    0    8G  0 disk
   ├─sda1                      8:1    0  512M  0 part /boot/efi
   ├─sda2                      8:2    0    1G  0 part /boot
   └─sda3                      8:3    0  6.5G  0 part
     └─ubuntu--vg-ubuntu--lv 253:0    0  6.5G  0 lvm  /
   sr0                        11:0    1 1024M  0 rom
  ```
- После увеличения диска в гипервизоре, ядро может не "увидеть" изменения, потому поможем ему командой `partprobe`
   ```
   # partprobe
   Warning: Not all of the space available to /dev/sda appears to be used, you can fix the GPT to use all of the space (an extra 2097152 blocks) or continue with the current setting?
   # lsblk
   NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
   loop0                       7:0    0   91M  1 loop /snap/core/6350
   loop1                       7:1    0 89.4M  1 loop /snap/core/6818
   sda                         8:0    0    9G  0 disk
   ├─sda1                      8:1    0  512M  0 part /boot/efi
   ├─sda2                      8:2    0    1G  0 part /boot
   └─sda3                      8:3    0  6.5G  0 part
     └─ubuntu--vg-ubuntu--lv 253:0    0  6.5G  0 lvm  /
   sr0                        11:0    1 1024M  0 rom
  ```
  Диск `sda` увеличился до 9 Гб
- Теперь увеличим последний раздел используя parted
   ```
   # parted /dev/sda
   ```
   Говорим parted, что единицы, с которыми работаем - сектора
   ```
   (parted) unit s
   ```
   Смотрим сколько свободного места
   ```
   (parted) print free
   ```
   > Может всплыть такое
   >
   > _Warning: Not all of the space available to /dev/sda appears to be used, you can fix the GPT to use all of the space (an extra 2097152 blocks) or continue with the current setting?_
   > пока не разобрался к чему это, потому соглашаюсь вводя `Fix`
   
   Получаем что-то вроде:
   ```
   ...
   Number  Start      End        Size       File system  Name  Flags
           34s        2047s      2014s      Free Space
    1      2048s      1050623s   1048576s   fat32              boot, esp
    2      1050624s   3147775s   2097152s   ext4
    3      3147776s   16777182s  13629407s
           16777183s  18874334s  2097152s   Free Space
   ```
   Расширяем третий раздел, до конца свободного места
   ```
   (parted) resizepart 3 18874334
   ```
   Смотрим чего получилось
   ```
   (parted) print free
   ...
   Number  Start     End        Size       File system  Name  Flags
           34s       2047s      2014s      Free Space
    1      2048s     1050623s   1048576s   fat32              boot, esp
    2      1050624s  3147775s   2097152s   ext4
    3      3147776s  18874334s  15726559s
   ```
   Раздел увеличен, можно выходить из parted
   ```
   (parted) quit
   ```
   Проверяем
   
   ```
   # lsblk
   ...
   └─sda3                      8:3    0  7.5G  0 part
   ...
   ```
   
   Как видно раздел `sda3` увеличился до 7.5 Гб
   
- Для увеличения физического тома нужно его имя, которое можно узнать командой `pvdisplay`
   ```
   # pvdisplay
   --- Physical volume ---
   PV Name               /dev/sda3
   VG Name               ubuntu-vg
   PV Size               <6.50 GiB / not usable 1.98 MiB
   Allocatable           yes (but full)
   PE Size               4.00 MiB
   Total PE              1663
   Free PE               0
   Allocated PE          1663
   PV UUID               td2oNT-O7zX-PdPy-GiE8-Nkvp-M4mN-tjFTTj
   ```
   Увеличиваем физический том
   ```
   pvresize /dev/sda3
   ```
   
   Проверяем   
   ```
   # pvdisplay
   ...
   PV Size               <7.50 GiB / not usable 1.98 MiB
   ...
   ```
- Для увеличения логического тома понадобится путь к нему (LV Path), состоящий из имени группы томов (VG Name) и имени самого логического тома (LV Name), их можно узнать командой `lvdisplay`
   ```
   # lvdisplay 
   --- Logical volume ---
   LV Path                /dev/ubuntu-vg/ubuntu-lv
   LV Name                ubuntu-lv
   VG Name                ubuntu-vg
   LV UUID                Cj8k2a-fPGO-eHW0-HcHu-xLkY-4uR3-qfqdgi
   LV Write Access        read/write
   LV Creation host, time ubuntu-server, 2019-05-15 15:24:45 +0000
   LV Status              available
   # open                 1
   LV Size                <6.50 GiB
   Current LE             1663
   Segments               1
   Allocation             inherit
   Read ahead sectors     auto
   - currently set to     256
   Block device           253:0
   ```
   Увеличение производится командой
   ```
   lvextend -l +100%FREE /dev/<volumeGroupName>/<logicalVolumeName>
   ```
   т.е. в нашем случае
   ```
   lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   ```
   Проверяем
   ```
   # lvdisplay 
   ...
   LV Size                <7.50 GiB
   ...
   ```
- Последним шагом станет увеличение размера файловой системы. Для ext4 производится командой
   ```
   resize2fs /dev/<volumeGroupName>/<logicalVolumeName>
   ```
   т.е. в нашем случае
   ```
   resize2fs /dev/ubuntu-vg/ubuntu-lv
   ```
   Проверяем
   ```
   # df -h
   ...
   /dev/mapper/ubuntu--vg-ubuntu--lv  7.4G  3.6G  3.4G  52% /
   ...
   ```
