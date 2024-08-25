# MySQL: бэкап и репликация
1. Развернуть стенд, предлагаемый в методическом материале;
2. Развернуть базу данных (БД) из дампа bet.dmp на мастер сервере и настроить так, чтобы реплицировались таблицы:<br/>
   |bookmaker|<br/>
   |competition|<br/>
   |market|<br/>
   |odds|<br/>
   |outcome|<br/>
3. Настроить GTID репликацию.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0, образа ubuntu/jammy64 версии 20240301.0.0. и mysql-server-8.0.39 <br/>
### Ход решения ###
1. Развёртывание стенда для выполнения домашнего задания:
```shell
vagrant up
```
2. Подготовка соответствующей инфраструктуры и развёртывание БД из дампа:
   - Установка mysql-server-8.0. на обоих серверах:
   ```shell
   apt update
   apt install mysql-server
   ```
   - Для того, чтобы в Ansible отрабатывали модули mysql, необходимо установить систему управления программными пакетами, написанными на Python и далее с её помощью инсталлировать требуемый пакет PyMySQL. Данный пакет представляет собой библиотеку на основе Python, которая позволяет программе Python подключиться к базе данных MySQL и взаимодействовать с ней, выполняя запросы для управления данными.
   ```shell
   apt install python3-pip
   pip3 install PyMySQL
   ```
   - Копирование конфигурационных файлов mysql:
   ```shell
   root@master:~# cp /vagrant/provisioning/files/conf.d/mysqld_master.cnf /etc/mysql/mysql.conf.d/mysqld.cnf

   root@slave:~# cp /vagrant/provisioning/files/conf.d/mysqld_slave.cnf /etc/mysql/mysql.conf.d/mysqld.cnf
   ```  
   - Перезапуск службы mysql:
   ```shell
   systemctl restart mysql
   ```
   - Установка пароля root в mysql:
   ```shell
   mysql -e \"ALTER USER 'root'@'localhost' IDENTIFIED WITH 'caching_sha2_password' BY 'Otus2024!'\"
   ```
   - Проверка отличия server-id на серверах:
   ```shell
   mysql> select @@server_id;
   +-------------+
   | @@server_id |
   +-------------+
   |           1 |
   +-------------+
   1 row in set (0.00 sec)

   mysql> select @@server_id;
   +-------------+
   | @@server_id |
   +-------------+
   |           2 |
   +-------------+
   1 row in set (0.00 sec)
   ```
   - Создание тестовой БД и загрузка в неё имеющегося дампа:
   ```shell
   mysql> CREATE DATABASE bet;
   Query OK, 1 row affected (0.00 sec)

   root@master:~# mysql -uroot -p -D bet < /vagrant/provisioning/files/bet.dmp

   mysql> use bet;
   Reading table information for completion of table and column names
   You can turn off this feature to get a quicker startup with -A

   Database changed
   mysql> show tables;
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
   - Создание пользователя для репликации и наделение его соответствующими правами:
   ```shell
   mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'OtusRepl2024!';
   mysql> SELECT user,host FROM mysql.user where user='repl';
   +------+------+
   | user | host |
   +------+------+
   | repl | %    |
   +------+------+
   1 row in set (0.00 sec)
   mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'OtusRepl2024!';
   ```
   - Подготовка дампа базы с игнорированием таблиц по заданию для переноса на слэйв сервер:
   ```shell
   root@master:~# mysqldump --all-databases --triggers --routines --master-data
   --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
   ```
   - Перенос дампа мастер сервера на слэйв сервер и проверка корретности переноса БД:
   ```shell
   mysql> SOURCE /tmp/master.sql
   mysql> SHOW DATABASES LIKE 'bet';
   +----------------+
   | Database (bet) |
   +----------------+
   | bet            |
   +----------------+
   1 row in set (0.07 sec)

   mysql> use bet;
   Reading table information for completion of table and column names
   You can turn off this feature to get a quicker startup with -A

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
   5 rows in set (0.01 sec)
   ```
   &ensp;&ensp;Вывод команды, свидетельствует об отсутствии таблиц v_same_event и events_on_demand в БД на слэйв сервере.<br/>
   - Подключение и запуск репликации на слэйв сервере:
   ```shell
   mysql> CHANGE REPLICATION SOURCE TO SOURCE_HOST = "192.168.11.150", SOURCE_PORT = 3306, SOURCE_USER = "repl", SOURCE_PASSWORD =  "OtusRepl2024!", SOURCE_AUTO_POSITION = 1;
   mysql> START REPLICA;
   mysql> SHOW REPLICA STATUS;
   *************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: 192.168.56.150
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: binlog.000004
          Read_Source_Log_Pos: 93619
               Relay_Log_File: slave-relay-bin.000003
                Relay_Log_Pos: 93829
        Relay_Source_Log_File: binlog.000004
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: bet.events_on_demand,bet.v_same_event
         ....
       Retrieved_Gtid_Set: 2bc7b2cb-62dd-11ef-93b2-0255be2b59e4:1-38
            Executed_Gtid_Set: 18bbce3a-62dd-11ef-8fee-0255be2b59e4:1-174,2bc7b2cb-62dd-11ef-93b2-0255be2b59e4:1-38
                Auto_Position: 1
   ```
   - Проверка работы репликации. На мастер сервере в БД 'bet' создаётся дополнительная запись '1xbet' в таблице 'bookmaker'. Данные изменения происходят и на слэйв сервере:<br/>
     Мастер-сервер
   ```shell
   mysql> USE bet;
   Database changed
   mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
   Query OK, 1 row affected (0.05 sec)

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
   &ensp;&ensp;&ensp;&ensp;Слэйв сервер
   ```shell
   mysql> USE bet;
   Database changed
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
&ensp;&ensp;Для автоматического конфигурирования инфраструктуры с помощью Ansible, подготовлен плейбук playbook.yml.
