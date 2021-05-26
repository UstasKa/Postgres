## Установка PostgreSQL 
### Задача:
- сделать GCE инстанс с Ubuntu 20.04
- поставить на нем Docker Engine
- сделать каталог /var/lib/postgres
- развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres
- развернуть контейнер с клиентом postgres
- подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
- подключится к контейнеру с сервером с ноутбука/комьютера вне инстансов GCP
- удалить контейнер с сервером
- создать его заново
- подключится снова из контейнера с клиентом к контейнеру с сервером
- проверить, что данные остались на месте
- оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами

### Решение
- сделать GCE инстанс с Ubuntu 20.04
- поставить на нем Docker Engine
```bash
apt update && apt upgrade -y
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```
- сделать каталог /var/lib/postgres
```bash
mkdir /var/lib/postgres
```
- развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres
```bash
docker run --name=postgre -d --restart=always -e POSTGRES_PASSWORD='otus' -e POSTGRES_USER='otus' -e POSTGRES_DB='otus' -v /var/lib/postgres:/var/lib/postgresql/data  -p 5432:5432 postgres:latest
```
- развернуть контейнер с клиентом postgres и
- подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
```bash
docker run -it --rm --link=postgre postgres:latest psql -h postgre -U otus otus
```
```sql
otus=# CREATE TABLE COMPANY(
otus(#    ID INT PRIMARY KEY     NOT NULL,
otus(#    NAME           TEXT    NOT NULL,
otus(#    AGE            INT     NOT NULL,
otus(#    ADDRESS        CHAR(50),
otus(#    SALARY         REAL
otus(# );
CREATE TABLE
otus=# \d
        List of relations
 Schema |  Name   | Type  | Owner
--------+---------+-------+-------
 public | company | table | otus
(1 row)

otus=# \d company;
                  Table "public.company"
 Column  |     Type      | Collation | Nullable | Default
---------+---------------+-----------+----------+---------
 id      | integer       |           | not null |
 name    | text          |           | not null |
 age     | integer       |           | not null |
 address | character(50) |           |          |
 salary  | real          |           |          |
Indexes:
    "company_pkey" PRIMARY KEY, btree (id)
```
- подключится к контейнеру с сервером с ноутбука/комьютера вне инстансов GCP
не забываем создать Firewall policy в GCP, открывающее порт 5432 и после этого подключаемся:
```bash
docker run -it --rm postgres:latest psql -h 34.66.42.199 -U otus otus
```
- удалить контейнер с сервером
```bash
docker stop postgre
docker rm postgre
```
- создать его заново
```bash
docker run --name=postgre -d --restart=always -e POSTGRES_PASSWORD='otus' -e POSTGRES_USER='otus' -e POSTGRES_DB='otus' -v /var/lib/postgres:/var/lib/postgresql/data  -p 5432:5432 postgres:latest
```
- подключится снова из контейнера с клиентом к контейнеру с сервером
```bash
docker run -it --rm --link=postgre postgres:latest psql -h postgre -U otus otus
```
- проверить, что данные остались на месте
```sql
otus=# \d company;
                  Table "public.company"
 Column  |     Type      | Collation | Nullable | Default
---------+---------------+-----------+----------+---------
 id      | integer       |           | not null |
 name    | text          |           | not null |
 age     | integer       |           | not null |
 address | character(50) |           |          |
 salary  | real          |           |          |
Indexes:
    "company_pkey" PRIMARY KEY, btree (id)
```
- оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами
### Комментарии
- обязательно использовать внешние папки для данных типа /var/lib/postgres;
- обязательно монтировать в папку /var/lib/postgresql/data иначе данные потеряются;
- Желательно использовать docker volumes;
- не забывать открывать порты на фаерволе в GCP.
