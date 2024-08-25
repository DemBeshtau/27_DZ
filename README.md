# MySQL: бэкап и репликация
1. Развернуть стенд, предлагаемый в методическом материале;
2. Развернуь базу данных (БД) из дампа bet.dmp на мастер сервере и настроить так, чтобы реплицировались таблицы:<br/>
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
