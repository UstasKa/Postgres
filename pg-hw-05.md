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

* autovacuum_max_workers (integer)
Задаёт максимальное число процессов автоочистки (не считая процесс, запускающий автоочистку), которые могут выполняться одновременно. По умолчанию это число равно трём. Задать этот параметр можно только при запуске сервера.

* autovacuum_naptime (integer)
Задаёт минимальную задержку между двумя запусками автоочистки для отдельной базы данных. Демон автоочистки проверяет базу данных через заданный интервал времени и выдаёт команды VACUUM и ANALYZE, когда это требуется для таблиц этой базы. Если это значение задаётся без единиц измерения, оно считается заданным в секундах. По умолчанию задержка равна одной минуте (1min). Этот параметр можно задать только в postgresql.conf или в командной строке при запуске сервера.

* autovacuum_vacuum_threshold (integer)
Задаёт минимальное число изменённых или удалённых кортежей, при котором будет выполняться VACUUM для отдельно взятой таблицы. Значение по умолчанию — 50 кортежей. Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_vacuum_insert_threshold (integer)
Задаёт число добавленных кортежей, при достижении которого будет выполняться VACUUM для отдельно взятой таблицы. Значение по умолчанию — 100 кортежей. При значении -1 процедура автоочистки не будет производить операции VACUUM с таблицами в зависимости от числа добавленных строк. Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_analyze_threshold (integer)
Задаёт минимальное число добавленных, изменённых или удалённых кортежей, при котором будет выполняться ANALYZE для отдельно взятой таблицы. Значение по умолчанию — 50 кортежей. Этот параметр можно задать только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_vacuum_scale_factor (floating point)
Задаёт процент от размера таблицы, который будет добавляться к autovacuum_vacuum_threshold при выборе порога срабатывания команды VACUUM. Значение по умолчанию — 0.2 (20% от размера таблицы). Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_vacuum_insert_scale_factor (floating point)
Задаёт процент от размера таблицы, который будет добавляться к autovacuum_vacuum_insert_threshold при выборе порога срабатывания команды VACUUM. Значение по умолчанию — 0.2 (20% от размера таблицы). Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_analyze_scale_factor (floating point)
Задаёт процент от размера таблицы, который будет добавляться к autovacuum_analyze_threshold при выборе порога срабатывания команды ANALYZE. Значение по умолчанию — 0.1 (10% от размера таблицы). Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако данное значение можно переопределить для избранных таблиц, изменив их параметры хранения.

* autovacuum_freeze_max_age (integer)
Задаёт максимальный возраст (в транзакциях) для поля pg_class.relfrozenxid некоторой таблицы, при достижении которого будет запущена операция VACUUM для предотвращения зацикливания идентификаторов транзакций в этой таблице. Заметьте, что система запустит процессы автоочистки для предотвращения зацикливания, даже если для всех других целей автоочистка отключена.

* autovacuum_multixact_freeze_max_age (integer)
Задаёт максимальный возраст (в мультитранзакциях) для поля pg_class.relminmxid таблицы, при достижении которого будет запущена операция VACUUM для предотвращения зацикливания идентификаторов мультитранзакций в этой таблице. Заметьте, что система запустит процессы автоочистки для предотвращения зацикливания, даже если для всех других целей автоочистка отключена.

* autovacuum_vacuum_cost_delay (floating point)
Задаёт задержку при превышении предела стоимости, которая будет применяться при автоматических операциях VACUUM. Если это значение задаётся без единиц измерения, оно считается заданным в миллисекундах. При значении -1 применяется обычная задержка vacuum_cost_delay. Значение по умолчанию — 2 миллисекунды. Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако его можно переопределить для отдельных таблиц, изменив их параметры хранения.

* autovacuum_vacuum_cost_limit (integer)
Задаёт предел стоимости, который будет учитываться при автоматических операциях VACUUM. При значении -1 (по умолчанию) применяется обычное значение vacuum_cost_limit. Заметьте, что это значение распределяется пропорционально среди всех работающих процессов автоочистки, если их больше одного, так что сумма ограничений всех процессов никогда не превосходит данный предел. Задать этот параметр можно только в postgresql.conf или в командной строке при запуске сервера. Однако его можно переопределить для отдельных таблиц, изменив их параметры хранения.

## Первый этап. Стандартные настройки Autovacuum
```sql
SELECT name, setting, context, short_desc FROM pg_settings WHERE category like '%Autovacuum%';
```
>количество живых и мертвых строчек и как давно заходил автовакуум
>SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';

| Name                                 |  setting  |  context   |  short_desc                                                                               |
|--------------------------------------|-----------|------------|-------------------------------------------------------------------------------------------|
|autovacuum                            | on        | sighup     | Starts the autovacuum subprocess.                                                         |
|autovacuum_analyze_scale_factor       | 0.1       | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples. |
|autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.                    |
|autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.                  |
|autovacuum_max_workers                | 3         | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.            |
|autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.             |
|autovacuum_naptime                    | 60        | sighup     | Time to sleep between autovacuum runs.                                                    |
|autovacuum_vacuum_cost_delay          | 2         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.                                        |
|autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.                              |
|autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.                       |
|autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.         |
|autovacuum_vacuum_scale_factor        | 0.2       | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.            |
|autovacuum_vacuum_threshold           | 50        | sighup     | Minimum number of tuple updates or deletes prior to vacuum.                               |

> TPS – это показатель пропускной способности базы данных, который показывает количество транзакций, обработанных базой данных за одну секунду. 

#### Запускаем pgbench 1
```bash
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 1200 -U postgres postgres
starting vacuum...end.
progress: 60.0 s, 945.8 tps, lat 8.454 ms stddev 4.542
progress: 120.0 s, 942.8 tps, lat 8.485 ms stddev 5.136
progress: 180.0 s, 904.4 tps, lat 8.845 ms stddev 5.290
progress: 240.1 s, 822.6 tps, lat 9.708 ms stddev 11.241
progress: 300.1 s, 587.6 tps, lat 13.616 ms stddev 22.996
progress: 360.1 s, 590.8 tps, lat 13.536 ms stddev 22.756
progress: 420.1 s, 595.3 tps, lat 13.441 ms stddev 22.494
progress: 480.1 s, 587.7 tps, lat 13.608 ms stddev 23.252
progress: 540.1 s, 569.4 tps, lat 14.049 ms stddev 24.035
progress: 600.1 s, 574.6 tps, lat 13.916 ms stddev 24.084
progress: 660.1 s, 576.3 tps, lat 13.878 ms stddev 24.024
progress: 720.1 s, 572.4 tps, lat 13.968 ms stddev 23.829
progress: 780.1 s, 579.3 tps, lat 13.808 ms stddev 23.280
progress: 840.1 s, 572.7 tps, lat 13.969 ms stddev 23.242
progress: 900.1 s, 589.9 tps, lat 13.561 ms stddev 22.625
progress: 960.1 s, 591.0 tps, lat 13.538 ms stddev 22.746
progress: 1020.1 s, 593.4 tps, lat 13.479 ms stddev 22.733
progress: 1080.1 s, 579.5 tps, lat 13.803 ms stddev 23.311
progress: 1140.1 s, 578.1 tps, lat 13.834 ms stddev 22.881
progress: 1200.1 s, 577.5 tps, lat 13.847 ms stddev 23.155
tps = 646.573241 (including connections establishing)
tps = 646.574573 (excluding connections establishing)
```
## Второй этап. Средние настройки Autovacuum
| Name                                 |  setting  |  context   |  short_desc                                                                               |
|--------------------------------------|-----------|------------|-------------------------------------------------------------------------------------------|
|autovacuum                            | on        | sighup     | Starts the autovacuum subprocess.                                                         |
|autovacuum_analyze_scale_factor       | 0.1       | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples. |
|autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.                    |
|autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.                  |
|autovacuum_max_workers                | 10        | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.            |
|autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.             |
|autovacuum_naptime                    | 120       | sighup     | Time to sleep between autovacuum runs.                                                    |
|autovacuum_vacuum_cost_delay          | 1         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.                                        |
|autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.                              |
|autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.                       |
|autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.         |
|autovacuum_vacuum_scale_factor        | 0.1       | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.            |
|autovacuum_vacuum_threshold           | 100       | sighup     | Minimum number of tuple updates or deletes prior to vacuum.                               |

#### Запускаем pgbench 2
```bash
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 1200 -U postgres postgres
starting vacuum...end.
```
## Второй этап. Высокие настройки Autovacuum
| Name                                 |  setting  |  context   |  short_desc                                                                               |
|--------------------------------------|-----------|------------|-------------------------------------------------------------------------------------------|
|autovacuum                            | on        | sighup     | Starts the autovacuum subprocess.                                                         |
|autovacuum_analyze_scale_factor       | 0.1       | sighup     | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples. |
|autovacuum_analyze_threshold          | 50        | sighup     | Minimum number of tuple inserts, updates, or deletes prior to analyze.                    |
|autovacuum_freeze_max_age             | 200000000 | postmaster | Age at which to autovacuum a table to prevent transaction ID wraparound.                  |
|autovacuum_max_workers                | 20        | postmaster | Sets the maximum number of simultaneously running autovacuum worker processes.            |
|autovacuum_multixact_freeze_max_age   | 400000000 | postmaster | Multixact age at which to autovacuum a table to prevent multixact wraparound.             |
|autovacuum_naptime                    | 240       | sighup     | Time to sleep between autovacuum runs.                                                    |
|autovacuum_vacuum_cost_delay          | 1         | sighup     | Vacuum cost delay in milliseconds, for autovacuum.                                        |
|autovacuum_vacuum_cost_limit          | -1        | sighup     | Vacuum cost amount available before napping, for autovacuum.                              |
|autovacuum_vacuum_insert_scale_factor | 0.2       | sighup     | Number of tuple inserts prior to vacuum as a fraction of reltuples.                       |
|autovacuum_vacuum_insert_threshold    | 1000      | sighup     | Minimum number of tuple inserts prior to vacuum, or -1 to disable insert vacuums.         |
|autovacuum_vacuum_scale_factor        | 0.7       | sighup     | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.            |
|autovacuum_vacuum_threshold           | 200       | sighup     | Minimum number of tuple updates or deletes prior to vacuum.                               |

#### Запускаем pgbench 3
```bash
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 1200 -U postgres postgres
starting vacuum...end.

```
### Выводы
1. При проведении тестирования необходимо помнить, что запуск pgbench грузит процессор более чем на 40%.
2. 
