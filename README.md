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

##### Настройка hot_standby репликации с использованием слотов 

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

##### Настройка резервного копирования

Настраиваем резервное копирование с помощью утилиты Barman. В документации Barman рекомендуется разворачивать Barman на отдельном сервере. В этом случае потребуется настроить доступы между серверами по SSH-ключам.

На хостах node1 и node2 необходимо установить утилиту barman-cli:
```
root@node1:~# apt install barman-cli
```
На хосте barman выполняем следующие настройки. Устанавливаем пакеты barman и postgresql-client:
```
root@barman:~# apt install barman-cli barman postgresql
```
Переходим в пользователя barman и генерируем ssh-ключ:
```
root@barman:~# su barman
barman@barman:/root$ cd 
barman@barman:~$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/barman/.ssh/id_rsa): 
Created directory '/var/lib/barman/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/barman/.ssh/id_rsa
Your public key has been saved in /var/lib/barman/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:EvwtFPKnsqBx/saq5mZ6Z5eGSXtIgae2+iCZ+aKI7e8 barman@barman
The key's randomart image is:
+---[RSA 4096]----+
|      . .        |
|     . o .       |
|   .  o o .      |
|  . o  + +       |
|  .oo.o S .      |
| +o=o. + .       |
|*..+.*..         |
|=+* B.B          |
|B%B*E*.          |
+----[SHA256]-----+
```
На хосте __node1__ переходим в пользователя postgres и генерируем ssh-ключ: 
```
root@node1:~# su postgres
postgres@node1:/root$ cd 
postgres@node1:~$ ssh-keygen -t rsa -b 4096
Generating public/private rsa key pair.
Enter file in which to save the key (/var/lib/postgresql/.ssh/id_rsa): 
Created directory '/var/lib/postgresql/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /var/lib/postgresql/.ssh/id_rsa
Your public key has been saved in /var/lib/postgresql/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:iek87rS0V28RMSvHcqNYe5LXdyXCmZzheM69TsfKwQs postgres@node1
The key's randomart image is:
+---[RSA 4096]----+
|                 |
|            +    |
|           * O   |
|       o .= ^ . .|
|      o So % * ..|
|     o  . * *.o.o|
|      *  . =E.+o+|
|     + +.   o+.= |
|     .=.   . .=  |
+----[SHA256]-----+
```
После генерации ключа, выводим содержимое файла ~/.ssh/id_rsa.pub:
```
postgres@node1:~$ cat ~/.ssh/id_rsa.pub
```
Копируем содержимое файла на сервер barman в файл __/var/lib/barman/.ssh/authorized_keys__

В psql создаём пользователя barman c правами суперпользователя:
```
postgres@node1:~$ psql
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE USER barman WITH REPLICATION Encrypted PASSWORD 'Otus2024!';
CREATE ROLE
postgres=# GRANT EXECUTE ON FUNCTION pg_start_backup(text, boolean, boolean) to barman;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_stop_backup() to barman;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_stop_backup(boolean, boolean) to barman;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_switch_wal() to barman;
GRANT
postgres=# GRANT EXECUTE ON FUNCTION pg_create_restore_point(text) to barman;
GRANT
postgres=# GRANT pg_read_all_settings TO barman;
GRANT ROLE
postgres=# GRANT pg_read_all_stats TO barman;
GRANT ROLE
```
В файл /etc/postgresql/14/main/pg_hba.conf добавляем разрешения для пользователя barman:
```
...
...
host    all           barman      192.168.56.13/32        scram-sha-256
host    replication   barman      192.168.56.13/32        scram-sha-256
```
Перезапускаем службу postgresql-14:
```
root@node1:~# systemctl restart postgresql
```
В psql создадим тестовую базу otus:
```
root@node1:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# CREATE DATABASE otus;
CREATE DATABASE
```
Cоздаём таблицу test в базе otus:
```
postgres=# \c otus;
otus=# CREATE TABLE test (id int, name varchar(30));
otus=# INSERT INTO test VALUES (1, 'evgenii');
INSERT 0 1
otus=# select * from test;
 id |  name   
----+---------
  1 | evgenii
(1 row)
```
Продолжаем настройку хоста barman. 

После генерации ключа, выводим содержимое файла ~/.ssh/id_rsa.pub:
```
barman@barman:~$ cat ~/.ssh/id_rsa.pub
```
Копируем содержимое файла на сервер node1 в файл /var/lib/postgresql/.ssh/authorized_keys. 

Находясь в пользователе barman создаём файл ~/.pgpass со следующим содержимым:
```
192.168.56.11:5432:*:barman:Otus2024!
```
В данном файле указываются реквизиты доступа для postgres. Через знак двоеточия пишутся следующие параметры: 
- ip-адрес
- порт postgres
- имя БД (* означает подключение к любой БД)
- имя пользователя
- пароль пользователя
- 
Файл должен быть с правами 600, владелец файла barman:
```
barman@barman:~$ chmod 600 ~/.pgpass
```
Проверяем возможность подключения к postgres-серверу:
```
barman@barman:~$ psql -h 192.168.56.11 -U barman -d postgres
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=> 
```
Проверяем репликацию: 
```
barman@barman:~$ psql -h 192.168.56.11 -U barman -c "IDENTIFY_SYSTEM" replication=1
      systemid       | timeline |  xlogpos  | dbname 
---------------------+----------+-----------+--------
 7410047013382606820 |        1 | 0/301A110 | 
(1 row)
```
Создаём файл /etc/barman.conf со следующим содержимым:
```
[barman]
#Указываем каталог, в котором будут храниться бекапы
barman_home = /var/lib/barman
#Указываем каталог, в котором будут храниться файлы конфигурации бекапов
configuration_files_directory = /etc/barman.d
#пользователь, от которого будет запускаться barman
barman_user = barman
#расположение файла с логами
log_file = /var/log/barman/barman.log
#Используемый тип сжатия
compression = gzip
#Используемый метод бекапа
backup_method = rsync
archiver = on
retention_policy = REDUNDANCY 3
immediate_checkpoint = true
#Глубина архива
last_backup_maximum_age = 4 DAYS
minimum_redundancy = 1
```
Владельцем файла должен быть пользователь barman
```
root@barman:~# chown barman:barman /etc/barman.conf
```
Создаём файл /etc/barman.d/node1.conf со следующим содержимым:
```
[node1]
#Описание задания
description = "backup node1"
#Команда подключения к хосту node1
ssh_command = ssh postgres@192.168.56.11
#Команда для подключения к postgres-серверу
conninfo = host=192.168.56.11 user=barman port=5432 dbname=postgres
retention_policy_mode = auto
retention_policy = RECOVERY WINDOW OF 7 days
wal_retention_policy = main
streaming_archiver=on
#Указание префикса, который будет использоваться как $PATH на хосте node1
path_prefix = /usr/pgsql-14/bin/
#настройки слота
create_slot = auto
slot_name = node1
#Команда для потоковой передачи от postgres-сервера
streaming_conninfo = host=192.168.56.11 user=barman
#Тип выполняемого бекапа
backup_method = postgres
archiver = off
```
Владельцем файла должен быть пользователь barman:
```
root@barman:~# chown barman:barman /etc/barman.d/node1.conf
```
На этом настройка бекапа завершена. Теперь проверим работу barman: 
```
root@barman:~# barman switch-wal node1
The WAL file 000000010000000000000003 has been closed on server 'node1'
root@barman:~# barman cron
Starting WAL archiving for server node1
root@barman:~# barman check node1
Server node1:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: FAILED (interval provided: 4 days, latest backup age: No available backups)
	backup minimum size: OK (0 B)
	wal maximum age: OK (no last_wal_maximum_age provided)
	wal size: OK (0 B)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: FAILED (have 0 backups, expected at least 1)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK (no system Id stored on disk)
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archiver errors: OK
```
После этого запускаем резервную копию:
```
root@barman:~# barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20240902T154026
Backup start at LSN: 0/4000148 (000000010000000000000004, 00000148)
Starting backup copy via pg_basebackup for 20240902T154026
WARNING: pg_basebackup does not copy the PostgreSQL configuration files that reside outside PGDATA. Please manually backup the following files:
	/etc/postgresql/14/main/postgresql.conf
	/etc/postgresql/14/main/pg_hba.conf
	/etc/postgresql/14/main/pg_ident.conf

Copy done (time: less than one second)
Finalising the backup.
This is the first backup for server node1
WAL segments preceding the current backup have been found:
	000000010000000000000003 from server node1 has been removed
Backup size: 41.8 MiB
Backup end at LSN: 0/6000000 (000000010000000000000005, 00000000)
Backup completed (start time: 2024-09-02 15:40:26.774565, elapsed time: less than one second)
Processing xlog segments from streaming for node1
	000000010000000000000004
WARNING: IMPORTANT: this backup is classified as WAITING_FOR_WALS, meaning that Barman has not received yet all the required WAL files for the backup consistency.
This is a common behaviour in concurrent backup scenarios, and Barman automatically set the backup as DONE once all the required WAL files have been archived.
Hint: execute the backup command with '--wait'
```
На этом процесс настройки бекапа закончен, для удобства команду `barman backup node1` требуется добавить в crontab. 

##### Проверка восстановления из бекапов

На хосте node1 в psql удаляем базы Otus:
```
postgres=# DROP DATABASE otus;
DROP DATABASE
postgres=# DROP DATABASE otus_test; 
DROP DATABASE
postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(3 rows)
```
Далее на хосте barman запустим восстановление:
```
root@barman:~# barman list-backup node1
node1 20240902T154026 - Mon Sep  2 12:40:27 2024 - Size: 41.8 MiB - WAL Size: 0 B
root@barman:~# barman recover node1 20240902T154026  /var/lib/postgresql/14/main/ --remote-ssh-comman "ssh postgres@192.168.56.11"
The authenticity of host '192.168.56.11 (192.168.56.11)' can't be established.
ED25519 key fingerprint is SHA256:pgmb0TceejFJrEokPQIlkCg+h0TbKTqy7wlrZ4p9sUo.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Starting remote restore for server node1 using backup 20240902T154026
Destination directory: /var/lib/postgresql/14/main/
Remote command: ssh postgres@192.168.56.11
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

WARNING
The following configuration files have not been saved during backup, hence they have not been restored.
You need to manually restore them in order to start the recovered PostgreSQL instance:

    postgresql.conf
    pg_hba.conf
    pg_ident.conf

Recovery completed (start time: 2024-09-02 15:45:49.427552, elapsed time: 7 seconds)

Your PostgreSQL server has been successfully prepared for recovery!
```
Далее на хосте node1 потребуется перезапустить postgresql-сервер и снова проверить список БД. Базы otus должны вернуться обратно:
```
root@node1:~# systemctl restart postgresql
root@node1:~# sudo -u postgres psql
could not change directory to "/root": Permission denied
psql (14.13 (Ubuntu 14.13-0ubuntu0.22.04.1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 otus      | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 otus_test | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(5 rows)
```
