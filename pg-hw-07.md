## Блокировки
---
### Задача
1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга? (Попробуйте воспроизвести такую ситуацию)

### Решение
0. Создаем новый кластер PostgresSQL 13.
```bash
apt update && apt upgrade -y && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt-get -y install postgresql && apt install unzip
systemctl status postgresql
pg_lsclusters
sudo -u postgres psql
```
1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
```sql
create table HW07(id int primary key, name varchar(10));
insert into HW07 values (1, 'Grisha');
insert into HW07 values (2, 'Vasya');
select * from HW07;
 id |  name
----+--------
  1 | Grisha
  2 | Vasya
(2 rows)
show deadlock_timeout ;
 deadlock_timeout
------------------
 1s
(1 row)
ALTER SYSTEM SET deadlock_timeout TO 200;
select pg_reload_conf();
show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)
```
### В первой сессии
```psql
begin;
select name FROM HW07 WHERE id = 1 FOR UPDATE;
select pg_sleep(20);
update HW07 SET name = 'V_Sess1' WHERE id = 2;
commit;
```
### Во второй сессии
```sql
begin;
select name FROM HW07 WHERE id = 2 FOR UPDATE;
update HW07 SET name = 'G_Sess2' WHERE id = 1;
commit;
```
### Получившийся результат
Получаем сообщение о блокировке и транзакция из первой сессии откатывается назад:
```sql
ERROR:  deadlock detected
DETAIL:  Process 12875 waits for ShareLock on transaction 521; blocked by process 12891.
Process 12891 waits for ShareLock on transaction 520; blocked by process 12875.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,28) in relation "hw07"
```
В логах видим сообщения:
```bash
cat /var/log/postgresql/postgresql-13-main.log | grep "deadlock detected" -A 10
2021-06-20 13:26:37.868 UTC [12875] postgres@postgres ERROR:  deadlock detected
2021-06-20 13:26:37.868 UTC [12875] postgres@postgres DETAIL:  Process 12875 waits for ShareLock on transaction 521; blocked by process 12891.
        Process 12891 waits for ShareLock on transaction 520; blocked by process 12875.
        Process 12875: update HW07 SET name = 'V_Sess1' WHERE id = 2;
        Process 12891: update HW07 SET name = 'G_Sess2' WHERE id = 1;
2021-06-20 13:26:37.868 UTC [12875] postgres@postgres HINT:  See server log for query details.
2021-06-20 13:26:37.868 UTC [12875] postgres@postgres CONTEXT:  while updating tuple (0,28) in relation "hw07"
2021-06-20 13:26:37.868 UTC [12875] postgres@postgres STATEMENT:  update HW07 SET name = 'V_Sess1' WHERE id = 2;
```
```sql
select * from HW07;
 id |  name
----+---------
  2 | Vasya
  1 | G_Sess2
(2 rows)
```
2. 
