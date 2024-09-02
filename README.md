### OTUS Linux Professional Lesson #46 | Subject: PostgreSQL репликация

#### Цель: Научиться настраивать репликацию и создавать резервные копии в СУБД PostgreSQL

#### Описание домашнего задания:
- настроить hot_standby репликацию с использованием слотов
- настроить правильное резервное копирование

#### Процедура выполнения домашнего задания:

Создаем три виртуальные машины:
```
# vagrant up
```

__Настройка hot_standby репликации с использованием слотов__

Устанавливаем postgres-server на хосты node1 и node2:
```
root@node1:~# apt install postgresql postgresql-contrib
root@node1:~# systemctl start postgresql
root@node1:~# systemctl enable postgresql
```
Переходим к настройке репликации на хосте __node1__. Заходим в psql:
```
root@node1:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.
```
В psql создаём пользователя replicator c правами репликации:
```
postgres=# CREATE USER replicator WITH REPLICATION Encrypted PASSWORD 'Otus2024!';
CREATE ROLE
```
В файле __/etc/postgresql/14/main/postgresql.conf__ указываем следующие параметры:
```
#Указываем ip-адреса, на которых postgres будет слушать трафик
listen_addresses = 'localhost, 192.168.56.11'
#Указываем порт postgres
port = 5432 
#Устанавливаем максимально 100 одновременных подключений
max_connections = 100
log_directory = 'log' 
log_filename = 'postgresql-%a.log' 
log_rotation_age = 1d 
log_rotation_size = 0 
log_truncate_on_rotation = on 
max_wal_size = 1GB
min_wal_size = 80MB
log_line_prefix = '%m [%p] ' 
#Указываем часовой пояс для Москвы
log_timezone = 'UTC+3'
timezone = 'UTC+3'
datestyle = 'iso, mdy'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8' 
lc_numeric = 'en_US.UTF-8' 
lc_time = 'en_US.UTF-8' 
default_text_search_config = 'pg_catalog.english'
#можно или нет подключаться к postgresql для выполнения запросов в процессе восстановления; 
hot_standby = on
#Включаем репликацию
wal_level = replica
#Количество планируемых слейвов
max_wal_senders = 3
#Максимальное количество слотов репликации
max_replication_slots = 3
#будет ли сервер slave сообщать мастеру о запросах, которые он выполняет.
hot_standby_feedback = on
#Включаем использование зашифрованных паролей
password_encryption = scram-sha-256
```
Настраиваем параметры подключения в файле __/etc/postgresql/14/main/pg_hba.conf__:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD
# "local" is for Unix domain socket connections only
local   all                  all                                       peer
# IPv4 local connections:
host    all                  all             127.0.0.1/32              scram-sha-256
# IPv6 local connections:
host    all                  all             ::1/128                   scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                peer
host    replication     all        127.0.0.1/32            scram-sha-256
host    replication     all        ::1/128                 scram-sha-256
host    replication replicator    192.168.56.11/32        scram-sha-256
host    replication replicator    192.168.56.12/32        scram-sha-256
```
Перезапускаем postgresql-server:
```
root@node1:~# systemctl restart postgresql
```
Настраиваем хост __node2__. 

Останавливаем postgresql-server:
```
root@node2:~# systemctl stop postgresql
```
В файле  __/etc/postgresql/14/main/postgresql.conf__ меняем параметр:
```
listen_addresses = 'localhost, 192.168.56.12'
```
Прежде чем Replica-сервер сможет начать реплицировать данные, нужно создать новую БД, идентичную Master-серверу. Для этого воспользуемся утилитой pg_basebackup. Она создаст бэкап с Master-сервера и скачает его на Replica-сервер. Эту операцию нужно выполнять от имени пользователя postgres, поэтому логинимся от него:
```
root@node2:~# su - postgres
```
Далее переходим в каталог с базой данных:
```
postgres@node2:~$ cd /var/lib/postgresql/14/
```
Удалим каталог с дефолтной БД и снова его создадим, но уже пустой:
```
postgres@node2:~$ rm -rf main; mkdir main; chmod go-rwx main
```
Теперь выгрузим БД с мастера. Для выполнения этой команды нужно будет ввести пароль от пользователя __replicator__, который мы задавали в самом начале настройки Master-сервера:
```
postgres@node2:~$ pg_basebackup -P -R -X stream -c fast -h 192.168.56.11 -U replicator -D ./main
```
В этой команде есть важный параметр -R. Он означает, что PostgreSQL-сервер также создаст пустой файл standby.signal. Несмотря на то, что файл пустой, само наличие этого файла означает, что этот сервер — реплика.

Возвращаемся в root-пользователя и запускаем PostgreSQL-сервер:
```
postgres@node2:~$ exit
root@node2:~# systemctl postgresql start
```
Проверяем работу репликации. На хосте __node1__ в psql создадим базу otus_test и выведем список БД:
```
root@node1:~# sudo -u postgres psql
postgres=# CREATE DATABASE otus_test;
CREATE DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)
```
На хосте __node2__ также в psql также проверим список БД:
```
root@node2:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(4 rows)
```
Также можно проверить репликацию другим способом. На хосте __node1__ в psql вводим команду:
```
postgres=# select * from pg_stat_replication;
```
На хосте node2 в psql вводим команду:
```
select * from pg_stat_wal_receiver;
```
Вывод обеих команд должен быть не пустым.

В случае выхода из строя master-хоста (node1), на slave-сервере (node2) в psql необхоимо выполнить команду select pg_promote();

Также можно создать триггер-файл. Если в дальнейшем хост node1 заработает корректно, то для восстановления его работы (как master-сервера) необходимо: 
- настроить сервер node1 как slave-сервер
- с помощью команды `select pg_promote();` перевести режим его работы в master
