## Журналы 
---
### Задача
1. Настройте выполнение контрольной точки раз в 30 секунд.
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу.
Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите
кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

### Решение
0. Создаем новый кластер PostgresSQL 13.
```bash
apt update && apt upgrade -y && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt-get -y install postgresql && apt install unzip
systemctl status postgresql
pg_lsclusters
sudo -u postgres psql
```
1. Настройте выполнение контрольной точки раз в 30 секунд.

```bash
echo "checkpoint_timeout = 30s" | sudo tee -a /etc/postgresql/13/main/postgresql.conf
systemctl restart postgresql
```
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

Сначала очистим статистику
```sql
SELECT pg_stat_reset_shared('bgwriter');
```
Запустим на 10 минут pgbench
```bash
sudo -u postgres pgbench -i postgres
sudo -u postgres pgbench -c8 -P 60 -T 600 -U postgres postgres
```
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```bash
du -m /var/lib/postgresql/13/main/pg_wal/*
16      /var/lib/postgresql/13/main/pg_wal/000000010000000000000022
16      /var/lib/postgresql/13/main/pg_wal/000000010000000000000023
16      /var/lib/postgresql/13/main/pg_wal/000000010000000000000024
16      /var/lib/postgresql/13/main/pg_wal/000000010000000000000025
 ```
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```sql
SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 21
checkpoints_req       | 0
checkpoint_write_time | 254719
checkpoint_sync_time  | 3671
buffers_checkpoint    | 193
buffers_clean         | 84121
maxwritten_clean      | 0
buffers_backend       | 390829
buffers_backend_fsync | 0
buffers_alloc         | 753583
stats_reset           | 2021-06-05 11:45:22.275452+00
```
> https://habr.com/ru/company/postgrespro/blog/460423/
- checkpoints_timed — по расписанию (по достижению checkpoint_timeout);
- checkpoints_req — по требованию (в том числе по достижению max_wal_size) (Большое значение checkpoint_req (по сравнению с checkpoints_timed) говорит о том, что контрольные точки происходят чаще, чем предполагалось.);

Важная информация о количестве записанных страниц:
- buffers_checkpoint — процессом контрольной точки;
- buffers_backend — обслуживающими процессами;
- buffers_clean — процессом фоновой записи.
Контрольные точки выполнялись по расписанию.

5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
```bash
sudo -u postgres pgbench -i buffer_temp
sudo -u postgres pgbench -P 1 -T 10 buffer_temp
tps = 665.106076 (including connections establishing)
tps = 665.321183 (excluding connections establishing)
```
```sql
ALTER SYSTEM SET synchronous_commit = off;
systemctl restart postgresql
```
```bash
sudo -u postgres pgbench -i buffer_temp
sudo -u postgres pgbench -P 1 -T 10 buffer_temp
tps = 1337.850104 (including connections establishing)
tps = 1338.243259 (excluding connections establishing)
```
В асинхронном режиме скорость увеличилась в два раза благодаря более эффективному алгоритму сбрасывания данных на диск.. Но при этом в случае сбоя есть рис потерять несколько последних транзакций (3 × wal_writer_delay).

6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу.
Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите
кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
```bash
pg_createcluster 13 testchecksums -- --data-checksums
pg_ctlcluster 13 testchecksums start
pg_lsclusters
Ver Cluster        Port Status Owner    Data directory                        Log file
13  main           5432 online postgres /var/lib/postgresql/13/main           /var/log/postgresql/postgresql-13-main.log
13  testchecksums  5433 online postgres /var/lib/postgresql/13/testchecksums  /var/log/postgresql/postgresql-13-testchecksums.log
sudo -u postgres psql
```
```sql
\c buffer_temp
SELECT pg_relation_filepath('test_text');
```
Остановим сервер и поменяем несколько байтов в странице (сотрем из заголовка LSN последней журнальной записи)
```bash
dd if=/dev/zero of=/var/lib/postgresql/13/main/base/16384/16410 oflag=dsync conv=notrunc bs=1 count=8
```
запустим сервер и попробуем сделать выборку из таблицы



```sql
проверяем размер кэша:
SELECT setting, unit FROM pg_settings WHERE name = 'shared_buffers'; 
уменьшаем его до 200 страниц и рестартуем кластер
ALTER SYSTEM SET shared_buffers = 200;
\q
```
```bash
pg_ctlcluster 13 main restart
```
