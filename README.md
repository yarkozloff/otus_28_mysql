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
### Подготовка master. После поднятия по vagrant up, провиженем машину ансиблом. Роль установит Percona-Server-server-57, скопирует необходимые мастеру конфиги, запустит службу mysql, скопирует дамп тестовой базы.

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
```
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
```
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected, 1 warning (0.00 sec)

Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию:
```
[root@master ~]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```
Далее снова выполняю роль ансибл, для того чтобы вытащить дамп на основную маишну.
### 
