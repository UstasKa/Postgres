## Задача:
- создать новый проект в Google Cloud Platform, например postgres2021-<yyyymmdd>, где yyyymmdd год, месяц и день вашего рождения 
  (имя проекта должно быть уникально на уровне GCP)
- дать возможность доступа к этому проекту пользователю ifti@yandex.ru с ролью Project Editor
- далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами
- добавить свой ssh ключ в GCE metadata
- зайти удаленным ssh (первая сессия), не забывайте про ssh-add
- поставить PostgreSQL
- зайти вторым ssh (вторая сессия)
- запустить везде psql из под пользователя postgres
- выключить auto commit
- сделать в первой сессии новую таблицу и наполнить ее данными
 create table persons(id serial, first_name text, second_name text);
 insert into persons(first_name, second_name) values('ivan', 'ivanov');
 insert into persons(first_name, second_name) values('petr', 'petrov');
 commit;
- посмотреть текущий уровень изоляции: show transaction isolation level
- начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
- в первой сессии добавить новую запись
 insert into persons(first_name, second_name) values('sergey', 'sergeev');
 - сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить первую транзакцию - commit;
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершите транзакцию во второй сессии
- начать новые но уже repeatable read транзакции - set transaction isolation level repeatable read;
- в первой сессии добавить новую запись
 insert into persons(first_name, second_name) values('sveta', 'svetova');
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить первую транзакцию - commit;
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- завершить вторую транзакцию
- сделать select * from persons во второй сессии
- видите ли вы новую запись и если да то почему?
- остановите виртуальную машину но не удаляйте ее
  ```sql
  \echo :AUTOCOMMIT
  \set AUTOCOMMIT OFF
  ```
  
##  Решение:
GCP VM - создана
PG - установлен
после \set AUTOCOMMIT OFF в read committed новые записи во второй сессии не видим, тк в первой сессии еще не закоммитили изменения
после коммита запись видна
в repeatable read запись во второй сессии не видна, тк в режиме repeatable read видны только те данные, 
которые были зафиксированы до начала транзакции, но не видны незафиксированные данные и изменения, 
произведённые другими транзакциями в процессе выполнения данной транзакции.
