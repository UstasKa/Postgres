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
CREATE EXTENSION pg_buffercache;
SELECT count(*) FROM pg_buffercache WHERE isdirty;
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

Можно включить логирование чекпоинтов в журнале событий:
```sql
ALTER SYSTEM SET log_checkpoints = on;
SELECT pg_reload_conf();
```
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
В асинхронном режиме скорость увеличилась в два раза благодаря более эффективному алгоритму сбрасывания данных на диск.. Но при этом в случае сбоя есть риск потерять несколько последних транзакций (3 × wal_writer_delay).

6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу.
Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите
кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
```bash
pg_createcluster 13 testchecksums -p 5555 -- --data-checksums
pg_ctlcluster 13 testchecksums start
sudo -u postgres psql -p 5555
SELECT * from pg_settings WHERE name ~ 'checksum';
CREATE TABLE checksums AS SELECT * FROM generate_series(1,100) AS g(n);
SELECT pg_relation_filepath('checksums');
\q
```
Остановим сервер и поменяем несколько байтов в странице (сотрем из заголовка LSN последней журнальной записи)
```bash
pg_ctlcluster 13 testchecksums stop
dd if=/dev/zero of=/var/lib/postgresql/13/main/base/13414/16384 oflag=dsync conv=notrunc bs=1 count=8
```
Запустим сервер и попробуем сделать выборку из таблицы
```bash
pg_ctlcluster 13 testchecksums11 start
sudo -u postgres psql -p 5555
select count(*) from checksums ;
 count
-------
   100
(1 row)
```
Что и почему произошло? 

data checksums указывает на необходимость проверки системой ввода/вывода контрольных сумм страниц для обнаружения повреждённых данных, так как по умолчанию проверка не производится. Включение проверки может в значительной мере оказать влияние на производительность. Когда проверка включена, производится вычисление контрольных сумм для всех объектов всех баз данных кластера поэтому гарнтируется целостность данных.

Как проигнорировать ошибку и продолжить работу?
> https://postgrespro.ru/docs/postgrespro/13/runtime-config-developer
* ignore_checksum_failure При обнаружении ошибок контрольных сумм при чтении Postgres Pro обычно сообщает об ошибке и прерывает текущую транзакцию. Если параметр ignore_checksum_failure включён, система игнорирует проблему (но всё же предупреждает о ней) и продолжает обработку. Это поведение может привести к краху, распространению или сокрытию повреждения данных и другим серьёзными проблемам. Однако, включив его, вы можете обойти ошибку и получить неповреждённые данные, которые могут находиться в таблице, если цел заголовок блока. Если же повреждён заголовок, будет выдана ошибка, даже когда этот параметр включён. По умолчанию этот параметр отключён (имеет значение off) и изменить его состояние может только суперпользователь.
* ignore_system_indexes Отключает использование индексов при чтении системных таблиц (при этом индексы всё же будут изменяться при записи в эти таблицы). Это полезно для восстановления работоспособности при повреждённых системных индексах. Этот параметр нельзя изменить после запуска сеанса.
* zero_damaged_pages При выявлении повреждённого заголовка страницы Postgres Pro обычно сообщает об ошибке и прерывает текущую транзакцию. Если параметр zero_damaged_pages включён, вместо этого система выдаёт предупреждение, обнуляет повреждённую страницу в памяти и продолжает обработку. Это поведение разрушает данные, а именно все строки в повреждённой странице. Однако, включив его, вы можете обойти ошибку и получить строки из неповреждённых страниц, которые могут находиться в таблице. Это бывает полезно для восстановления данных, испорченных в результате аппаратной или программной ошибки. Обычно включать его следует только тогда, когда не осталось никакой другой надежды на восстановление данных в повреждённых страницах таблицы. Обнулённые страницы не сохраняются на диск, поэтому прежде чем выключать этот параметр, рекомендуется пересоздать проблемные таблицы или индексы. По умолчанию этот параметр отключён (имеет значение off) и изменить его состояние может только суперпользователь.

