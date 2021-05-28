## MVCC, vacuum и autovacuum. 
---
### Задача
- создать GCE инстанс типа e2-medium и SSD 10GB
- установить на него PostgreSQL 13 с дефолтными настройками
- применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
- выполнить pgbench -i postgres
- запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres
- дать отработать до конца
- зафиксировать среднее значение tps в последней ⅙ части работы
- а дальше настроить autovacuum максимально эффективно
- так чтобы получить максимально ровное значение tps на горизонте часа
- отчет лучше в экселе с графиками

### Решение
- создать GCE инстанс типа e2-medium и SSD 10GB и установить на него PostgreSQL 13 с дефолтными настройками
```bash
apt update && apt upgrade -y && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt-get -y install postgresql && apt install unzip
systemctl status postgresql
```
- применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
```bash
Делаем бэкап конфига на всякий:
cp /etc/postgresql/13/main/postgresql.conf /etc/postgresql/13/main/postgresql.conf_old
```
Задаем парамерты согласно заданию в /etc/postgresql/13/main/postgresql.conf:
```bash
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
## некорректный параметр work_mem = 6553kB заменил на work_mem = 6MB
- выполнить pgbench -i postgres
