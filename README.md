# mysql_replication
####  Описание    
[Percona-server](https://www.percona.com/software/mysql-database/percona-server)- это бесплатная, открытая (open source), усовершенствованная альтернатива MySQL с полной обратной совместимостью, которая предоставляет превосходную производительность, масштабируемость и широкие возможности    
#### Задача   
Развернуть базу из дампа bet.dmp и настроить репликацию.    
Базу развернуть на мастере и настроить чтобы реплицировались таблицы:   
```
| bookmaker |
| competition |
| market |
| odds |
| outcome   
```
#### Vagrant стенд    
измененнный fork [стенда](https://gitlab.com/otus_linux/stands-mysql)   
`Vagrant up` поднимает машину `master`- ip 192.168.112.60 и `slave`- ip 192.168.112.61 на которые устанавливает `Percona-Server-server-57`    
основной конфиг Percona `/etc/my.cnf`   
include `/etc/my.cnf.d/`    
Дата файлы в `/var/lib/mysql`   
#### Настройка Master   
При установке Percona автоматически генерирует пароль для пользователя root и кладет его в
файл `/var/log/mysqld.log` который можно вывести следующей командой:    
**`cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'`**    
Подключаемся к mysql и меняем пароль для доступа к полному функционалу:   
**`mysql -uroot -p'Password'`**    
где Password - сгенерированный при установке пароль.    
**`mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';`**   
Репликация настроена с использованием `GTID`, подробнее в [документации](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html)   
Атрибут `server-id` на мастер-сервере должен обязательно отличаться от `server-id` слейв-сервера. Проверить какая переменная установлена в текущий
момент можно следующим образом:   
**`mysql> SELECT @@server_id;`**    
```
+---------------------+   
| @@server_id |   
+---------------------+   
| 1 |   
+---------------------+   
```     
Статус GTID:    
**`mysql> SHOW VARIABLES LIKE 'gtid_mode';`**   
```   
+-----------------------+---------+
| Variable_name | Value |
+-----------------------+--------+
| gtid_mode | ON |
+-----------------------+--------+
```   
Создадим тестовую базу `bet` и загрузим в нее дамп и проверим:    
**`mysql> CREATE DATABASE bet;`**   
`Query OK, 1 row affected (0.00 sec)`   
**`[root@master ~] mysql -uroot -p -D bet < /vagrant/bet.dmp`**   
где [bet.dmp](https://github.com/Hanafeevrus/mysql_replication/blob/master/bet.dmp)   
сменим базу данных:   
**`mysql> USE bet;`**   
**`mysql> SHOW TABLES;`**   
```
+-----------------------------+
| Tables_in_bet |
+-----------------------------+
| bookmaker |
| competition |
| events_on_demand |
| market |
| odds |
| outcome |
| v_same_event |
+------------------------------+    
```   
Создадим пользователя для репликации и даем ему права на эту самую репликацию:    
**`mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!passforrepl1234';`**   
**`mysql> SELECT user,host FROM mysql.user where user='repl';`**    
```
+------+-----------+
| user | host |
+------+-----------+
| repl | % |
+------+-----------+    
```   
**`mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!passforrepl1234';`**   
Дампим базу для последующего залива на слэйв и игнорируем таблицу по заданию:   
**`[root@master ~] mysqldump --all-databases --triggers --routines --master-data
--ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql`**   
где `master.sql` -дамп    
На этом настройка Master-а завершена. Файл дампа нужно залить на слейв.   

#### Настройка Slave    
Изменяем автоматически сгенерированный пароль при установке Percona:    
**`cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'`**    
**`mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';`**     
Проверяем server-id:    
**`mysql> SELECT @@server_id;`**    
```
+---------------------+
| @@server_id |
+---------------------+
| 2 |
+---------------------+   
```    
В конфиге `/etc/my.cnf.d/05-binlog.cnf` таблицы игнорирования при репликации.   
```
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event   
```   
Заливаем дамп мастера и убеждаемся что база есть и она без лишних таблиц:   
Переместить ранее созданную БД можно утилитой `vagrant scp`   
**`mysql> SOURCE /mnt/master.sql`**   
**`mysql> SHOW DATABASES LIKE 'bet';`**   
```
+-----------------------+
| Database (bet) |
+-----------------------+
| bet |
+-----------------------+   
```   
**`mysql> USE bet;`**   
**`mysql> SHOW TABLES;`**   
```
+---------------------+
| Tables_in_bet |
+---------------------+
| bookmaker |
| competition |
| market |
| odds |
| outcome |
+---------------------+ # видим что таблиц v_same_event и events_on_demand нет    
```       
В конфиг [`my.cnf`](https://github.com/Hanafeevrus/mysql_replication/blob/master/conf/my.cnf) добавлено игнорирование ошибок:   
`slave-skip-errors = 1007,1396`   
Подклячаем и запускаем слейв:   
**`mysql> CHANGE MASTER TO MASTER_HOST = "192.168.112.60", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!passforrepl1234", MASTER_AUTO_POSITION = 1;`**   
**`mysql> START SLAVE;`**   
**`mysql> SHOW SLAVE STATUS\G`**    
[вывод](https://github.com/Hanafeevrus/mysql_replication/blob/master/show%20slave%20status%5CG)
#### Проверка репликации    
Добавить на master'e:   
**`mysql> USE bet;`**   
**`mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');`** 
**`mysql> SELECT * FROM bookmaker;`**   
```
+----+---------------------------+
| id | bookmaker_name |
+----+---------------------------+
| 1 | 1xbet |
| 4 | betway |
| 5 | bwin |
| 6 | ladbrokes |
| 3 | unibet |
+----+--------------------------+   
```   
Проверить на slave'e:   
**`mysql> SELECT * FROM bookmaker;`**   
![show slave status](https://github.com/Hanafeevrus/mysql_replication/blob/master/show%20slave%20status.png)    
