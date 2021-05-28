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
с параметром work_mem = 6553KB кластер отказался стартовать из-за моей опечатки. заменил на work_mem = 6553kB. после этого запустился без проблем
```bash
root@pg-hw-05:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 down   postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
root@pg-hw-05:~# tail -f /var/log/postgresql/postgresql-13-main.log
2021-05-27 17:52:58.793 GMT [13122] LOG:  invalid value for parameter "work_mem": "6553KB"
2021-05-27 17:52:58.793 GMT [13122] HINT:  Valid units for this parameter are "B", "kB", "MB", "GB", and "TB".
2021-05-27 17:52:58.793 UTC [13122] FATAL:  configuration file "/etc/postgresql/13/main/postgresql.conf" contains errors
pg_ctl: could not start server
Examine the log output.
2021-05-27 18:05:42.884 GMT [13218] LOG:  invalid value for parameter "work_mem": "6553KB"
2021-05-27 18:05:42.884 GMT [13218] HINT:  Valid units for this parameter are "B", "kB", "MB", "GB", and "TB".
2021-05-27 18:05:42.885 UTC [13218] FATAL:  configuration file "/etc/postgresql/13/main/postgresql.conf" contains errors
pg_ctl: could not start server
```
> Почитать про Автовакуум можно здесь:
> https://postgrespro.ru/docs/postgrespro/13/runtime-config-autovacuum
- выполнить pgbench -i postgres
