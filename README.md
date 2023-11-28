# Задание № 3. #

- Добавить в Vagrantfile еще дисков.
- Собрать RAID-массив уровня 0/5/10 на выбор.
- Прописать собранный массив в конфиг, чтобы он собирался при загрузке
- Сломать/починить RAID.
- Создать GPT-таблицу и 5 разделов поверх массива, смонтировать их в системе.

**Ход решения**

Список блочных устройств (дисков).

[root@otus-task3 ~]# **lsblk**

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  250M  0 disk
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0   40G  0 disk
└─sdf1   8:81   0   40G  0 part /

**Создание RAID**

Создание RAID 5 из 5 дисков.

**mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{a,b,c,d,e}**

Посмотреть текущее состояние RAID'а

[root@otus-task3 ~]# **cat /proc/mdstat**
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sde[5] sdd[3] sdc[2] sdb[1] sda[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
unused devices: <none>

**Создание конфигурационного файла mdadm.conf**

mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

**Сломать/починить RAID**

Назничим диск sde сломаным.
mdadm /dev/md0 --fail /dev/sde

И "вытащим" его из серевера.
mdadm /dev/md0 --remove /dev/sde

А теперь добавим новый исправный диск.
mdadm /dev/md0 --add /dev/sde

Ребилд идёт успешно.
[root@otus-task3 ~]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid5 sde[5] sdd[3] sdc[2] sdb[1] sda[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [UUUU_]
      [========>............]  recovery = 44.6% (113792/253952) finish=0.3min speed=6693K/sec

**Создать GPT-таблицу и 5 разделов, смонтировать их в системе**
Создать таблицу разделов GPT.
parted -s /dev/md0 mklabel gpt
Создать сами разделы.
parted /dev/md0 mkpart primary ext4 0% 20%
parted /dev/md0 mkpart primary ext4 20% 40%
parted /dev/md0 mkpart primary ext4 40% 60%
parted /dev/md0 mkpart primary ext4 60% 80%
parted /dev/md0 mkpart primary ext4 80% 100%
Создать на этих разделах файловую систему ext4.
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mkdir -p /raid/part{1,2,3,4,5}
И смонтировать всё это безобразие.
for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
Посмотреть, что получилось.
[root@otus-task3 ~]# lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdb         8:16   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdc         8:32   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdd         8:48   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sde         8:64   0  250M  0 disk
└─md0       9:0    0  992M  0 raid5
  ├─md0p1 259:0    0  196M  0 md    /raid/part1
  ├─md0p2 259:1    0  198M  0 md    /raid/part2
  ├─md0p3 259:2    0  200M  0 md    /raid/part3
  ├─md0p4 259:3    0  198M  0 md    /raid/part4
  └─md0p5 259:4    0  196M  0 md    /raid/part5
sdf         8:80   0   40G  0 disk
└─sdf1      8:81   0   40G  0 part  /

И, в заключении, поправить файл /etc/fstab, чтобы разделы монтировалсь после перезагрузки. Добавим 5 строк:

/dev/md0p1                      /raid/part1             ext4            defaults        1 2
/dev/md0p2                      /raid/part2             ext4            defaults        1 2
/dev/md0p3                      /raid/part3             ext4            defaults        1 2
/dev/md0p4                      /raid/part4             ext4            defaults        1 2
/dev/md0p5                      /raid/part5             ext4            defaults        1 2
