# mysql_replication
####  description   
[Percona-server](https://www.percona.com/software/mysql-database/percona-server)- it is a free, open source, advanced MySQL alternative with full backward compatibility that provides superior performance, scalability and features    
#### Task   
Expand the database from the dump bet.dmp and configure replication.
Deploy the database on the wizard and configure the tables to replicate:    
```
| bookmaker |
| competition |
| market |
| odds |
| outcome   
```
#### Vagrant stand    
modified fork [stand](https://gitlab.com/otus_linux/stands-mysql)   
`Vagrant up` raises the machine `master`- ip 192.168.112.60 и `slave`- ip 192.168.112.61 on which it installs `Percona-Server-server-57`    
Percona main config `/etc/my.cnf`   
include `/etc/my.cnf.d/`    
data files `/var/lib/mysql`   
#### Master configurations    
When installed, Percona automatically generates a password for the root user and puts it in
the file `/ var / log / mysqld.log` which can be displayed with the following command:    
**`cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'`**    
We connect to mysql and change the password to access the full functionality:   
**`mysql -uroot -p'Password'`**    
where Password is the password generated during installation.   
**`mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';`**   
Replication is configured using `GTID`, more details in [documentation](https://dev.mysql.com/doc/refman/5.6/en/replication-gtids-concepts.html)   
The server-id attribute on the master server must be different from the server-id slave server. You can check which variable is currently set as follows:   
**`mysql> SELECT @@server_id;`**    
```
+---------------------+   
| @@server_id |   
+---------------------+   
| 1 |   
+---------------------+   
```     
GTID Status:    
**`mysql> SHOW VARIABLES LIKE 'gtid_mode';`**   
```   
+-----------------------+---------+
| Variable_name | Value |
+-----------------------+--------+
| gtid_mode | ON |
+-----------------------+--------+
```   
Let's create the test base `bet` and load the dump into it and check:   
**`mysql> CREATE DATABASE bet;`**   
`Query OK, 1 row affected (0.00 sec)`   
**`[root@master ~] mysql -uroot -p -D bet < /vagrant/bet.dmp`**   
where [bet.dmp](https://github.com/Hanafeevrus/mysql_replication/blob/master/bet.dmp)   
change the DB:    
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
Let's create a user for replication and give him rights to this very replication:   
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
Dump the base for the subsequent gulf on the slave and ignore the table by assignment:    
**`[root@master ~] mysqldump --all-databases --triggers --routines --master-data
--ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql`**   
where `master.sql` - dump    
This completes the setup of the Master. The dump file must be uploaded to the slave.    

#### Slave configuration.   
Change the automatically generated password when installing Percona:  
**`cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'`**    
**`mysql > ALTER USER USER() IDENTIFIED BY 'YourStrongPassword';`**     
check the server-id:    
**`mysql> SELECT @@server_id;`**    
```
+---------------------+
| @@server_id |
+---------------------+
| 2 |
+---------------------+   
```    
In the `/ etc / my.cnf.d / 05-binlog.cnf` config, the replication ignore table.   
```
replicate-ignore-table=bet.events_on_demand
replicate-ignore-table=bet.v_same_event   
```   
Fill the master dump and make sure that the database is there and it is without extra tables:
The previously created database can be moved using the `vagrant scp` utility.   
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
+---------------------+ # we see that there are no tables v_same_event and events_on_demand   
```       
Error ignoring has been added to the config [`my.cnf`](https://github.com/Hanafeevrus/mysql_replication/blob/master/conf/my.cnf):   
`slave-skip-errors = 1007,1396`   
Podlyaklyaem and run the slave:   
**`mysql> CHANGE MASTER TO MASTER_HOST = "192.168.112.60", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!passforrepl1234", MASTER_AUTO_POSITION = 1;`**   
**`mysql> START SLAVE;`**   
**`mysql> SHOW SLAVE STATUS\G`**    
[conclusion](https://github.com/Hanafeevrus/mysql_replication/blob/master/show%20slave%20status%5CG)
#### check replication   
Add to master:   
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
Check for slave:   
**`mysql> SELECT * FROM bookmaker;`**   
![show slave status](https://github.com/Hanafeevrus/mysql_replication/blob/master/show%20slave%20status.png)    
You can also check the changes in the log files:    
**`mysqlbinlog /var/lib/mysql/slave-relay-bin.000007`**   
[slave-relay-bin.000007](https://github.com/Hanafeevrus/mysql_replication/blob/master/slave-relay-bin.000007) - binary log file.
Added lines   
`INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet')`   
![log fragment](https://github.com/Hanafeevrus/mysql_replication/blob/master/log_slave.png)
