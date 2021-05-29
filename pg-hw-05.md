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
progress: 60.0 s, 955.9 tps, lat 8.365 ms stddev 5.532
progress: 120.0 s, 993.0 tps, lat 8.055 ms stddev 5.131
progress: 180.0 s, 980.3 tps, lat 8.160 ms stddev 5.090
progress: 240.0 s, 963.5 tps, lat 8.302 ms stddev 5.476
progress: 300.0 s, 976.6 tps, lat 8.191 ms stddev 5.186
progress: 360.0 s, 679.1 tps, lat 11.778 ms stddev 19.003
progress: 420.0 s, 598.5 tps, lat 13.365 ms stddev 22.667
progress: 480.0 s, 609.5 tps, lat 13.124 ms stddev 22.722
progress: 540.0 s, 608.4 tps, lat 13.147 ms stddev 22.390
progress: 600.0 s, 606.4 tps, lat 13.193 ms stddev 22.508
progress: 660.0 s, 612.8 tps, lat 13.055 ms stddev 21.686
progress: 720.0 s, 592.6 tps, lat 13.498 ms stddev 22.565
progress: 780.0 s, 605.5 tps, lat 13.207 ms stddev 22.604
progress: 840.0 s, 596.8 tps, lat 13.404 ms stddev 22.994
progress: 900.0 s, 597.7 tps, lat 13.383 ms stddev 22.450
progress: 960.0 s, 602.5 tps, lat 13.278 ms stddev 22.211
progress: 1020.0 s, 604.8 tps, lat 13.226 ms stddev 21.691
progress: 1080.0 s, 615.5 tps, lat 12.997 ms stddev 21.622
progress: 1140.0 s, 610.5 tps, lat 13.099 ms stddev 21.493
progress: 1200.0 s, 607.3 tps, lat 13.172 ms stddev 22.037
tps = 700.857840 (including connections establishing)
tps = 700.859344 (excluding connections establishing)
```
## Третий этап. Высокие настройки Autovacuum
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
progress: 60.0 s, 963.9 tps, lat 8.295 ms stddev 5.418
progress: 120.0 s, 966.4 tps, lat 8.277 ms stddev 5.269
progress: 180.0 s, 973.7 tps, lat 8.215 ms stddev 5.160
progress: 240.0 s, 958.6 tps, lat 8.345 ms stddev 5.193
progress: 300.0 s, 964.0 tps, lat 8.298 ms stddev 5.215
progress: 360.0 s, 648.3 tps, lat 12.339 ms stddev 20.375
progress: 420.0 s, 625.4 tps, lat 12.786 ms stddev 20.776
progress: 480.0 s, 629.2 tps, lat 12.714 ms stddev 21.053
progress: 540.0 s, 611.6 tps, lat 13.073 ms stddev 21.786
progress: 600.0 s, 628.2 tps, lat 12.735 ms stddev 21.310
progress: 660.0 s, 614.6 tps, lat 13.012 ms stddev 21.896
progress: 720.0 s, 610.6 tps, lat 13.101 ms stddev 22.334
progress: 780.0 s, 610.1 tps, lat 13.112 ms stddev 22.456
progress: 840.0 s, 607.0 tps, lat 13.177 ms stddev 22.444
progress: 900.0 s, 610.7 tps, lat 13.096 ms stddev 22.419
progress: 960.0 s, 604.9 tps, lat 13.223 ms stddev 22.484
progress: 1020.0 s, 610.5 tps, lat 13.101 ms stddev 22.100
progress: 1080.0 s, 604.0 tps, lat 13.240 ms stddev 22.658
progress: 1140.0 s, 600.5 tps, lat 13.319 ms stddev 22.720
progress: 1200.0 s, 600.2 tps, lat 13.328 ms stddev 22.696
tps = 702.115887 (including connections establishing)
tps = 702.117358 (excluding connections establishing)
```
### Примечание
- При проведении тестирования необходимо помнить, что запуск pgbench грузит процессор более чем на 40%.

## Результат

![TPS](https://user-images.githubusercontent.com/80751278/120077668-332a7980-c0b4-11eb-9989-25fde43758b3.JPG)

