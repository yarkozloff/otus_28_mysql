# Репликация mysql
## Описание ДЗ
В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:
| bookmaker |
| competition |
| market |
| odds |
| outcome

- Настроить GTID репликацию
Варианты которые принимаются к сдаче
- рабочий вагрантафайл
- скрины или логи SHOW TABLES
- конфиги
- пример в логе изменения строки и появления строки на реплике
## Пошаговая инструкция выполнения домашнего задания
### Подготовка среды
Версии необходимого ПО:
```
root@yarkozloff:/otus# hostnamectl | grep "Operating System"
  Operating System: Ubuntu 20.04.3 LTS
root@yarkozloff:/otus# vboxmanage --version
6.1.26_Ubuntur145957
root@yarkozloff:/otus# vagrant --version
Vagrant 2.2.19
```
Для развертывания нам понадобится 2 машины master и slave. Используем для них локальный бокс centos7 (при необходимости загрузки образа из облака Vagrant - в Vagrantfile изменить config.vm.box на centos/7).
### Подготовка master. 
После поднятия по vagrant up, провиженем машину ансиблом. Роль установит Percona-Server-server-57, скопирует необходимые мастеру конфиги, запустит службу mysql, скопирует дамп тестовой базы.

Далее подключаемся на маишну. При установке Percona автоматически генерирует пароль для пользователя root и кладет его в файл /var/log/mysqld.log. Достаем пароль и меняем его подключившись к mysql:
```
[root@master ~]# cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}'
B.N0pyc(tpOy
[root@master ~]#  mysql -uroot -p'B.N0pyc(tpOy'
...
mysql > ALTER USER USER() IDENTIFIED BY 'vs.55.YYDRR';
```
При настройке репликации с использованием GTID на мастере параметр server_id должен отличаться от слэйва:
```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)
```
Убеждаемся что GTID включен:
```
mysql>  SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.01 sec)
```
Создадим тестовую базу bet и загрузим в нее дамп и проверим:
```
mysql> CREATE DATABASE bet;
Query OK, 1 row affected (0.00 sec)
...
[root@master ~]# mysql -uroot -p -D bet < /etc/my.cnf.d/bet.dmp
...
mysql> USE bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> SHOW TABLES;
+------------------+
| Tables_in_bet    |
+------------------+
| bookmaker        |
| competition      |
| events_on_demand |
| market           |
| odds             |
| outcome          |
| v_same_event     |
+------------------+
7 rows in set (0.00 sec)
```

Создадим пользователя для репликации и даем ему права на эту самую репликацию:
```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected, 1 warning (0.00 sec)
```
Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:
```
[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```
Далее снова выполняю роль ансибл, для того чтобы вытащить дамп на основную маишну.
### Подготовка slave.
Выполняем роль для развертывания на slave компонента Percona-Server-server-57. Также перенесется дамп базы из мастера, скопируются конфиги. В конфиге 01-base.cnf правим server_id отличный от мастера, а в 05-binlog.cnf раскоментируем строки, которые будут игнорировать таблицы по заданию. 

Подключаемся к машине.
Заливаем дамп мастера, убеждаемся что база есть, и она без лишних таблиц:
```
mysql> SOURCE master.sql/master.sql/master/root/master.sql
mysql> SHOW DATABASES LIKE 'bet';
+----------------+
| Database (bet) |
+----------------+
| bet            |
+----------------+
1 row in set (0.00 sec)
mysql> USE bet;
Database changed
mysql> SHOW TABLES;
+---------------+
| Tables_in_bet |
+---------------+
| bookmaker     |
| competition   |
| market        |
| odds          |
| outcome       |
+---------------+
5 rows in set (0.07 sec)
```
### Подключаем слэйв
```
mysql> CHANGE MASTER TO MASTER_HOST = "192.168.50.10", MASTER_PORT = 3306, MASTER_USER = "repl", MASTER_PASSWORD = "!OtusLinux2018", MASTER_AUTO_POSITION = 1;
Query OK, 0 rows affected, 2 warnings (0.11 sec)

mysql> START SLAVE;
Query OK, 0 rows affected (0.02 sec)

mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.50.10
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 120423
               Relay_Log_File: slave-relay-bin.000002
                Relay_Log_Pos: 627
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: No
```
Видим, что слэйв не запускается. Смотрим ошибки: Last_Error: Error 'Can't create database 'bet'; database exists' on query. Default database: 'bet'. Query: 'CREATE DATABASE bet'

Пробуем остановить слэйв. Смотрим статус мастера из слэйва:
```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
Смотрим статус мастера из мастера:
```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                         |
+------------------+----------+--------------+------------------+-------------------------------------------+
| mysql-bin.000002 |   120423 |              |                  | c8523631-142c-11ed-a983-5254004d77d3:1-42 |
```
Пробуем установить идентификатор GTID в слэйв:
```
mysql> set global GTID_PURGED="c8523631-142c-11ed-a983-5254004d77d3:1-42";
```
Стартуем слэйв, смотрим статус:
```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.50.10
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 120423
               Relay_Log_File: slave-relay-bin.000003
                Relay_Log_Pos: 470
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
```
Видим, что слэйв работает, а в Replicate_Ignore_Table перечислены таблицы, которые игнорируются.
### Проверим репликацию в действии на мастере:
```
mysql> USE bet;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql>  INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
Query OK, 1 row affected (0.07 sec)

mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.00 sec)
```
На слэйве:
```
mysql> SELECT * FROM bookmaker;
+----+----------------+
| id | bookmaker_name |
+----+----------------+
|  1 | 1xbet          |
|  4 | betway         |
|  5 | bwin           |
|  6 | ladbrokes      |
|  3 | unibet         |
+----+----------------+
5 rows in set (0.08 sec)
```
Попытался найти bin логи mysql, но на слэйве нашел их не в не очень корректном виде:
```
[root@slave ~]# cat /var/lib/mysql/
auto.cnf                ib_logfile0             mysql.sock.lock         slave-relay-bin.000003
bet/                    ib_logfile1             performance_schema/     slave-relay-bin.index
ca-key.pem              ibtmp1                  private_key.pem         slave-slow.log
ca.pem                  master.info             public_key.pem          sys/
client-cert.pem         mysql/                  relay-log.info          xb_doublewrite
client-key.pem          mysql-bin.000001        server-cert.pem
ib_buffer_pool          mysql-bin.index         server-key.pem
ibdata1                 mysql.sock              slave-relay-bin.000002
[root@slave ~]# cat /var/lib/mysql/slave-relay-bin.000003
```
![image](https://user-images.githubusercontent.com/69105791/182964720-cbdb549b-3904-4be5-86a6-e84cec5d7254.png)

Но, тут видно последнее изменение строки
