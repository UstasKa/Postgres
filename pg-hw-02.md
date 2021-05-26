## Физический уровень PostgreSQL 
---
### Задача:
- создайте виртуальную машину c Ubuntu 20.04 LTS в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a
- поставьте на нее PostgreSQL через sudo apt
- проверьте что кластер запущен через sudo -u postgres pg_lsclusters
- зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
  postgres=# create table test(c1 text);
  postgres=# insert into test values('1');
  \q
- остановите postgres например через sudo -u postgres pg_ctlcluster 13 main stop
- создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
- добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
- проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, 
  в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
- сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
- перенесите содержимое /var/lib/postgres/13 в /mnt/data - mv /var/lib/postgresql/13 /mnt/data
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
- напишите получилось или нет и почему
- задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/13/main который надо поменять и поменяйте его
- напишите что и почему поменяли
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
- напишите получилось или нет и почему
- зайдите через через psql и проверьте содержимое ранее созданной таблицы

### задание со звездочкой *: 
- не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, 
  перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так
  чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

### Решение:
в GCP создаем ВМ и устанавливаем PG
создаем диск на 10гб и подключаем его к ВМ с PG
останавливаем PG, подключаем в систему диск, переносим данные:
```bash
systemctl stop postgresql
systemctl status postgresql
lsblk
parted /dev/sdb mklabel gpt
parted -a opt /dev/sdb mkpart primary ext4 0% 100%
lsblk
mkfs.ext4 -L datapartition /dev/sdb1
mkdir -p /mnt/data
mount -o defaults /dev/sdb1 /mnt/data
vi /etc/fstab
mount -a
df -h
chown -R postgres:postgres /mnt/data/
ls -la /mnt/
ls -la /var/lib/postgresql/13/
mv /var/lib/postgresql/13/ /mnt/data/
```
пытаемся запустить PG и получаем ошибку:
```bash
sudo -u postgres pg_ctlcluster 13 main start
Error: /var/lib/postgresql/13/main is not accessible or does not exist
```
находим параметр "data_directory" в конфигурационном файле и прописываем новый путь к данным:
```bash
grep -n data_directory /etc/postgresql/13/main/postgresql.conf
    41:data_directory = '/var/lib/postgresql/13/main'               # use data in another directory
```
запускаем PG и проверяем, что данные перенесены успешно:
```sql
sudo -u postgres pg_ctlcluster 13 main start
sudo -u postgres psql
\c iso
\d persons
select * from persons
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sidor      | sidorov
  5 | sveta      | svetova
(5 rows)
```
### Решение задание со звездочкой *
в GCP создаем вторую ВМ postgres-12-second и устанавливаем на нее PG:
```bash
apt update && apt upgrade -y && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt-get -y install postgresql && apt install unzip
systemctl stop postgresql
systemctl status postgresql
rm -rf /var/lib/postgresql
```
останавливаем первый сервер, отключаем диск и подключаем его к этой ВМ:
```bash
lsblk
mkdir /var/lib/postgresql
ls -la /var/lib/postgresql
chown -R postgres:postgres /var/lib/postgresql/
mount /dev/sdb1 /var/lib/postgresql/
df -h
ls -la /var/lib/postgresql
ls -la /var/lib/postgresql/13/
systemctl start postgresql
systemctl status postgresql
```
```sql 
sudo -u postgres psql
\c iso
\d persons
select * from persons
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sidor      | sidorov
  5 | sveta      | svetova
(5 rows)
```
* в fstab прописывать "/dev/sdb1 /var/lib/postgresql ext4 defaults 0 2" не стал, тк это для разового теста
