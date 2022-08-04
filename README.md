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
