### Домашнее задание к уроку "Установка и настройка PostgreSQL"


Для этого урока была использована виртуальная машина (ВМ), созданная и настроенная ранее для выполнения "домашки" к уроку 1, а именно: Ubuntu 22.04 LTS, с установленным PostgreSQL версий 14 и 15 и возможностью подключения по SSH.

Подключаемся к машине и проверяем, что кластер с нужной нам версий СУБД работает:
`sudo -u postgres pg_lsclusters`:
```
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main-%Y-%m-%d_%H%M%S.log
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main-%Y-%m-%d_%H%M%S.log
```
Всё в порядке, можно продолжать.

Подключимся к кластеру `sudo -iu postgres psql -d postgres` и выполним некоторое количество произвольных действий:
```
CREATE DATABASE "HomeWork 03";
\c "HomeWork 03"
CREATE TABLE proof AS SELECT now();
SELECT * FROM proof;
             now
------------------------------
 2024-04-20 10:34:25.02824+03
(1 row)
```

Затем отключимся от БД и выключим виртуалку, чтобы к ней можно было добавить ещё один диск.

Используя интерфейс командной сроки, создадим ещё один диск: `vboxmanage createmedium disk --filename /media/hadrian/STORAGE/VMs/PGDBA/NewDisk.vhd --size 10240 --format VHD --variant Fixed`.
Дождавшись завершения сего процесса, подключаем новый диск к нашей ВМ: `vboxmanage storageattach PGDBA --storagectl SATA --port 1 --type hdd --medium /media/hadrian/STORAGE/VMs/PGDBA/NewDisk.vhd`, после чего запускаем её и подключаемся по SSH.
Запустив `lsblk`, проверяем, что диск виден в системе:
```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 63.9M  1 loop /snap/core20/2182
loop1    7:1    0 63.9M  1 loop /snap/core20/2264
loop2    7:2    0   87M  1 loop /snap/lxd/27037
loop3    7:3    0   87M  1 loop /snap/lxd/27948
loop4    7:4    0 40.4M  1 loop /snap/snapd/20671
loop5    7:5    0 39.1M  1 loop /snap/snapd/21184
sda      8:0    0   50G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0   50G  0 part /
sdb      8:16   0   10G  0 disk
```
Естественно, что он пустой, поэтому с помощью `sudo fdisk /dev/sdb` создадим раздел на диске, а с помощью `sudo mkfs.ext4 /dev/sdb1` - его отформатируем.

Далее создадим директорию `sudo mkdir -p /mnt/data` и выставим ей нужные права и соответствующего владельца:
```
sudo chown postgres: -R /mnt/data
sudo chmod 700 /mnt/data
```

С помощью `sudo blkid` выясним UUID диска, после чего добавим строчку
```
UUID="0e172353-5eab-4cbc-b28f-65234f8125cf"    /mnt/data       ext4    defaults        0       1
```
в /etc/fstab и перезагрузимся.

После перезагрузки проверим, что диск в процессе загрузки успешно подключился: `df -kh`
```
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1.1M  196M   1% /run
/dev/sda2        49G  7.2G   40G  16% /
tmpfs           982M  1.1M  981M   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
/dev/sdb1       9.8G   46M  9.2G   1% /mnt/data
tmpfs           197M  4.0K  197M   1% /run/user/1001

```

Всё в порядке, движемся дальше. Останавливаем кластер и перемещаем директорию с БД:
```
sudo systemctl stop postgresql@15-main
sudo mv -v /var/lib/postgresql/15 /mnt/data
```

Запускаем кластер `sudo systemctl start postgresql@15-main` и ожидаемо получаем ошибку. Подробное изучение системных журналов показывает точную причину: `Error: /var/lib/postgresql/15/main is not accessible or does not exist`. Правим параметр `data_directory` конфигурационного файла `/etc/postgresql/15/main/postgresql.conf` и задаём ему значение `/mnt/data/15/main`, после чего кластер запускается без ошибок. Проверяем:
```
$ sudo -u postgres psql -d 'HomeWork 03' -c 'SELECT * FROM proof;'
             now
------------------------------
 2024-04-20 10:34:25.02824+03
(1 row)

```

На этом основная часть задания выполнена. Переходим ко второй части.

Поскольку в запасниках нашлась относительно нестарая ВМ, было решено использовать её.  Туда был установлен Postgres 15 версии, а доступ по SSH уже работал и ранее. Выключаем обе виртуалки, чтобы можно было переподключить внешний диск:
```
vboxmanage storageattach PGDBA --storagectl SATA --port 1 --type hdd --medium none
vboxmanage storageattach "Ubuntu 18.04 LTS" --storagectl SATA --port 1 --type hdd --medium /media/hadrian/STORAGE/VMs/PGDBA/NewDisk.vhd
```

После переподключения запускаем вторую виртуалку и подключаемся к ней с помощью SSH. 
Далее создаём нужную директорию, раздаём права, назначаем владельца и монтируем папку с БД. Поскольку нам не требуется навсегда подключать базу, обойдёмся без внесения изменений в /etc/fstab:
```
sudo mkdir -p /mnt/data
sudo chmod 700 /mnt/data
sudo chown postgres: /mnt/data
sudo mount /dev/sdb1 /mnt/data
```
`lsblk` нам показывает, что диск успешно пристыковался в нужное место:
```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 116.7M  1 loop /snap/core/14447
loop1    7:1    0   104M  1 loop /snap/core/16928
sda      8:0    0    25G  0 disk
├─sda1   8:1    0     1M  0 part
└─sda2   8:2    0    25G  0 part /
sdb      8:16   0    10G  0 disk
└─sdb1   8:17   0    10G  0 part /mnt/data
``` 
Правим postgresql.conf: `data_directory = '/mnt/data/15/main'`, после чего выполняем перезапуск СУБД. Вместо успешного рестарта базы неожиданно получаем ошибку: `Error: /mnt/data/15/main is not accessible or does not exist`. Странно всё это. Проверяем:
```
sudo ls -la /mnt/data/15/main
total 92
drwx------ 19 114 120 4096 Apr 20 11:16 ./
drwxr-xr-x  3 114 120 4096 Apr  4 19:58 ../
-rw-------  1 114 120    3 Apr  4 19:58 PG_VERSION
drwx------  6 114 120 4096 Apr 20 10:33 base/
-rw-------  1 114 120   68 Apr 20 11:15 current_logfiles
drwx------  2 114 120 4096 Apr 20 11:16 global/
drwx------  2 114 120 4096 Apr  4 19:58 pg_commit_ts/
drwx------  2 114 120 4096 Apr  4 19:58 pg_dynshmem/
drwx------  4 114 120 4096 Apr 20 11:16 pg_logical/
drwx------  4 114 120 4096 Apr  4 19:58 pg_multixact/
drwx------  2 114 120 4096 Apr  4 19:58 pg_notify/
drwx------  2 114 120 4096 Apr  4 19:58 pg_replslot/
drwx------  2 114 120 4096 Apr  4 19:58 pg_serial/
drwx------  2 114 120 4096 Apr  4 19:58 pg_snapshots/
drwx------  2 114 120 4096 Apr 20 11:16 pg_stat/
drwx------  2 114 120 4096 Apr  4 19:58 pg_stat_tmp/
drwx------  2 114 120 4096 Apr  4 19:58 pg_subtrans/
drwx------  2 114 120 4096 Apr  4 19:58 pg_tblspc/
drwx------  2 114 120 4096 Apr  4 19:58 pg_twophase/
drwx------  3 114 120 4096 Apr  4 19:58 pg_wal/
drwx------  2 114 120 4096 Apr  4 19:58 pg_xact/
-rw-------  1 114 120   88 Apr  4 19:58 postgresql.auto.conf
-rw-------  1 114 120  120 Apr 20 11:15 postmaster.opts
```
Вместо ожидаемых наменований `postgres` в качестве пользователя и группы, владеющих файлами и директориями, мы видим числовые идентификаторы. Выясняем id пользователя и группы `postgres` на этой машине:
```
$ id postgres
uid=111(postgres) gid=115(postgres) groups=115(postgres),113(ssl-cert)
```
Ситуация проясняется: поскольку на второй машине в разное время было устанавлено множество разнообразного ПО, uid и gid пользователя `postgres` стали отличаться от таковых на первой виртуалке. Хоть мы и до монтирования диска установили нужного владельца директории (`sudo chown postgres: /mnt/data`), это не помогло, так как после монтирования оказалось, что файлы и директории на внешнем диске имеют свои uid и gid. Поэтому повторим установку владельцев, но теперь уже для всей нижележащей структуры: `sudo chown postgres: -R /mnt/data`. 
Запускаем кластер: `sudo -u postgres pg_ctlcluster 15 main start`. Ошибок нет. Неужели получилось? Выполняем: `sudo -u postgres psql -d 'HomeWork 03' -c 'SELECT * FROM proof;'`
В ответ нам выдаётся какое-то несущественное предупреждение о несовпадении версий сортировки. Но это всё ерунда, так как удаётся прочесть данные из таблицы:
```
WARNING:  database "HomeWork 03" has a collation version mismatch
DETAIL:  The database was created using collation version 2.35, but the operating system provides version 2.27.
HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE "HomeWork 03" REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
             now
------------------------------
 2024-04-20 10:34:25.02824+03
(1 row)
```
Различие версий COLLATION объясняется тем, что база была создана на машине с версией Ubuntu 22.04, а затем диск был подключен к ВМ с Ubuntu 18.04. Для полного счастья изменим версию сортировки:
```
$ sudo -u postgres psql -d 'HomeWork 03' -c 'ALTER DATABASE "HomeWork 03" REFRESH COLLATION VERSION;'
WARNING:  database "HomeWork 03" has a collation version mismatch
DETAIL:  The database was created using collation version 2.35, but the operating system provides version 2.27.
HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE "HomeWork 03" REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
NOTICE:  changing version from 2.35 to 2.27
ALTER DATABASE
```
Теперь проверочный запрос выполняется уже без этого предупреждения:
```
$  sudo -u postgres psql -d 'HomeWork 03' -c 'SELECT * FROM proof;'                                             now
------------------------------
 2024-04-20 10:34:25.02824+03
(1 row)
```
Вторая часть задания выполнена.