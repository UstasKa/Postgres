### Задача
1. создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)
2. зайдите в созданный кластер под пользователем postgres
3. создайте новую базу данных testdb
4. зайдите в созданную базу данных под пользователем postgres
5. создайте новую схему testnm
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
7. вставьте строку со значением c1=1
8. создайте новую роль readonly
9. дайте новой роли право на подключение к базе данных testdb
10. дайте новой роли право на использование схемы testnm
11. дайте новой роли право на select для всех таблиц схемы testnm
12. создайте пользователя testread с паролем test123
13. дайте поль readonly пользователю testread
14. зайдите под пользователем testread в базу данных testdb
15. сделайте select * from t1;
16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
17. напишите что именно произошло в тексте домашнего задания
18. у вас есть идеи почему? ведь права то дали?
19. посмотрите на список таблиц
20. подсказка в шпаргалке под пунктом 20
21. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
22. вернитесь в базу данных testdb под пользователем postgres
23. удалите таблицу t1
24. создайте ее заново но уже с явным указанием имени схемы testnm
25. вставьте строку со значением c1=1
26. зайдите под пользователем testread в базу данных testdb
27. сделайте select * from testnm.t1;
28. получилось?
29. есть идеи почему? если нет - смотрите шпаргалку
30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
31. сделайте select * from testnm.t1;
32. получилось?
33. есть идеи почему? если нет - смотрите шпаргалку
31. сделайте select * from testnm.t1;
32. получилось?
33. ура!
34. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
35. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
36. есть идеи как убрать эти права? если нет - смотрите шпаргалку
37. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
38. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39. расскажите что получилось и почему

### 
1. создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)
```bash
apt update && apt upgrade -y && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt-get -y install postgresql && apt install unzip
systemctl status postgresql
```
2. зайдите в созданный кластер под пользователем postgres
```bash
sudo -u postgres psql
```
3. создайте новую базу данных testdb
```sql
create database testdb;
```
4. зайдите в созданную базу данных под пользователем postgres
```sql
\c testdb
```
5. создайте новую схему testnm
```sql
create schema testnm;
```
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
```sql
create table t1(c1 integer);
```
7. вставьте строку со значением c1=1
```sql
insert into t1 values(1);
```
8. создайте новую роль readonly
```sql
create role readonly;
```
9. дайте новой роли право на подключение к базе данных testdb
```sql
grant CONNECT on DATABASE testdb to readonly;
```
10. дайте новой роли право на использование схемы testnm
```sql
grant USAGE on SCHEMA testnm to readonly;
```
11. дайте новой роли право на select для всех таблиц схемы testnm
```sql
grant SELECT on ALL tables in schema testnm to readonly;
```
12. создайте пользователя testread с паролем test123
```sql
create user testread with password 'test123';
```
13. дайте роль readonly пользователю testread
```sql
grant readonly TO testread;
```
14. зайдите под пользователем testread в базу данных testdb
```sql
testdb=# \c testdb testread
FATAL:  Peer authentication failed for user "testread"
Previous connection kept
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}
 testread  |                                                            | {readonly}
```
Зайти под testread не получилось, тк пользовательне имеет прав на логон.
в документации нашел:
https://postgrespro.ru/docs/postgresql/13/role-membership
В стандарте SQL есть чёткое различие между пользователями и ролями. При этом пользователи, в отличие от ролей, не наследуют права автоматически. Такое поведение может быть получено в PostgreSQL, если для ролей, используемых как роли в стандарте SQL, устанавливать атрибут INHERIT, а для ролей-пользователей в стандарте SQL — атрибут NOINHERIT. Однако в PostgreSQL все роли по умолчанию имеют атрибут INHERIT. Это сделано для обратной совместимости с версиями до 8.1, в которых пользователи всегда могли использовать права групп, членами которых они являются.
Атрибуты роли LOGIN, SUPERUSER, CREATEDB и CREATEROLE можно рассматривать как особые права, но они никогда не наследуются как обычные права на объекты базы данных. Чтобы ими воспользоваться, необходимо переключиться на роль, имеющую этот атрибут, с помощью команды SET ROLE. Продолжая предыдущий пример, можно установить атрибуты CREATEDB и CREATEROLE для роли admin. Затем при входе с ролью joe, получить доступ к этим правам будет возможно только после выполнения SET ROLE admin.
Поэтому сделал так:
```sql
ALTER ROLE testread WITH LOGIN;
```
переключиться на пользователя не получилось.
только так:
```bash
\q
sudo -u postgres psql -h localhost -U testread -d testdb
```
15. сделайте select * from t1;
```sql
select * from t1;
```
16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
```sql
testdb=> select * from t1;
ERROR:  permission denied for table t1
testdb=>
```
17-21. напишите что именно произошло в тексте домашнего задания.
Таблица создана в схеме public, а не testnm и прав на public для роли readonly не давали

22. вернитесь в базу данных testdb под пользователем postgres
```sql
\q
sudo -u postgres psql
\c testdb
```
23. удалите таблицу t1
```sql
drop table t1;
```
24. создайте ее заново но уже с явным указанием имени схемы testnm
```sql
create table testnm.t1(c1 integer);
```
25. вставьте строку со значением c1=1
```sql
insert into testnm.t1 values(1);
```
26. зайдите под пользователем testread в базу данных testdb
27. сделайте select * from testnm.t1;
28. получилось?
не получилось
```sql
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
29. есть идеи почему? если нет - смотрите шпаргалку
мы пересоздавали таблицу, поэтому прав нет.
30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
```sql
\c testdb postgres;
alter default privileges in schema testnm grant select on tables to readonly;
\c testdb testread;
```
31. сделайте select * from testnm.t1;
32. получилось?
нет
33. есть идеи почему? если нет - смотрите шпаргалку
потому что alter default будет действовать для новых таблиц а grant select on all tables in schema testnm TO readonly отработал только для существующих на тот момент времени. надо сделать снова или grant select или пересоздать таблицу
```sql
grant select on all tables in schema testnm TO readonly;
```
31. сделайте select * from testnm.t1;
32. получилось?
Да, тепер все получилось
34. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
35. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
36. есть идеи как убрать эти права? если нет - смотрите шпаргалку
Посмотрел шпаргалку:
это все потому что search_path указывает в первую очередь на схему public. А схема public создается в каждой базе данных по умолчанию. И grant на все действия в этой схеме дается роли public. А роль public добавляется всем новым пользователям. Соответсвенно каждый пользователь может по умолчанию создавать объекты в схеме public любой базы данных, ес-но если у него есть право на подключение к этой базе данных. Чтобы раз и навсегда забыть про роль public - а в продакшн базе данных про нее лучше забыть - выполните следующие действия:
```sql
\c testdb postgres;
revoke create on schema public from public;
revoke all on database testdb from public;
\c testdb testread;
```
### Выводы
- Не простая тема с правами и требует детального изучения, если стоит задача гранулярной раздачи прав;
- Может возникнуть проблема с правами при удалении и добавлении таблиц заново;
- На почитать https://postgrespro.ru/docs/postgresql/13/user-manag

