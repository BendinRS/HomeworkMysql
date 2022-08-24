# Домашняя работа "MySQL: Backup + Репликация"


<details>
<summary>Описание</summary>

В материалах приложены ссылки на вагрант для репликации и дамп базы bet.dmp  
Базу развернуть на мастере и настроить так, чтобы реплицировались таблицы:  
| bookmaker |
| competition |
| market |
| odds |
| outcome
  
Настроить GTID репликацию  
x  
варианты которые принимаются к сдаче:  
+ рабочий вагрантафайл  
+ скрины или логи SHOW TABLES  
+ конфиги  
+ пример в логе изменения строки и появления строки на реплике  

</details>



<details>
<summary>Цели</summary>

снимать резервные копии;  
восстанавливать базу после сбоя;  
настраивать master-slave репликацию.  

</details>


# Практика

## Подготовка стенда

+ Скрипт [test.sh](test.sh) запускает разворачивание и настройку ансиблом ВМ  

+ При установке Percona автоматически генерирует паролþ длā полþзователā root и кладет его в
файл /var/log/mysqld.log  
    Заранее через ансибл вытащил его в файл  
    ```
      - name: pass step_1 save
    shell: cat /var/log/mysqld.log | grep 'root@localhost:' | awk '{print $11}' > ./password_mysql
    ```

+ Подключаемся к mysql и меняем пароль для доступа к полному функционалу:  

![Alt text](/screenshots/password.png?raw=true "full")

## Репликация

+ Атрибут server-id на мастер-сервере должен обязательно отличаться от server-id слейв-сервера. Проверить какая переменная установлена в текущий момент можно следующим образом:  

```
mysql> SELECT @@server_id;
+-------------+
| @@server_id |
+-------------+
|           1 |
+-------------+
1 row in set (0.00 sec)

```

Конфиг был проброшен ансиблом через темплейт  

+ Убеждаемся что включен GTID:

```
mysql> SHOW VARIABLES LIKE 'gtid_mode';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| gtid_mode     | ON    |
+---------------+-------+
1 row in set (0.00 sec)

```
+ Создадим тестовую базу bet и загрузим в нее дамп и проверим наличие таблиц:

![Alt text](/screenshots/bases.png?raw=true "check")

+ Создадим пользователя для репликации и даем ему права на эту самую репликацию:  

```
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '!OtusLinux2018';
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT user,host FROM mysql.user where user='repl';
+------+------+
| user | host |
+------+------+
| repl | %    |
+------+------+
1 row in set (0.00 sec)

```

+ Дампим базу для последующего залива на слэйв и игнорируем таблицы по заданию: 

```
[root@master vagrant]# mysqldump --all-databases --triggers --routines --master-data --ignore-table=bet.events_on_demand --ignore-table=bet.v_same_event -uroot -p > master.sql
```

+ Настройка Master-а завершена. Файл дампа передан на слейв.

## Настройка слейв

+ повторяем процедуры с подключением к mysql и проверкам

![Alt text](/screenshots/slave.png?raw=true "check")

+ Раскомментируем в /etc/my.cnf.d/05-binlog.cnf строки:  

```
#replicate-ignore-table=bet.events_on_demand
#replicate-ignore-table=bet.v_same_event
```

+ Заливаем дамп мастера и убеждаемся что база есть и она без лишних таблиц:  

![Alt text](/screenshots/slavetestbd.png?raw=true "check")

+ Подключаем, запускаем и проверяем слейв:

![Alt text](/screenshots/slave_itog.png?raw=true "check")

Слейв не запустился. При попытке подгрузить верный глобальный ид - ошибка

```
mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+----------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                      |
+------------------+----------+--------------+------------------+----------------------------------------+
| mysql-bin.000001 |      398 |              |                  | 9863dad1-23a5-11ed-8a2d-5254004d77d3:1 |
+------------------+----------+--------------+------------------+----------------------------------------+
1 row in set (0.00 sec)

mysql> set global GTID_PURGED="9863dad1-23a5-11ed-8a2d-5254004d77d3:1-74";
ERROR 1840 (HY000): @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_EXECUTED is empty.

```

Помог сайт
https://avdeo.com/tag/error-1840-hy000-global-gtid_purged-can-only-be-set-when/

Мы не можем присвоить пустое значение переменной gtid_executed, потому что она доступна только для чтения и с тех пор, как мы восстановили резервную копию, gtid_executed обновляется до момента, когда этот экземпляр выполнил транзакции.  

Единственный способ исправить это - использовать сброс мастера на ведомом устройстве. Что я и сделал.  

![Alt text](/screenshots/finish.png?raw=true "check")

## Скрины работы слейва

+ Логи
![Alt text](/screenshots/workslave.png?raw=true "check")
+ Мастер
```
mysql> USE bet;
mysql> INSERT INTO bookmaker (id,bookmaker_name) VALUES(1,'1xbet');
mysql> SELECT * FROM bookmaker;

```
![Alt text](/screenshots/master1.png?raw=true "check")

+ Слейв

![Alt text](/screenshots/slave2.png?raw=true "check")











