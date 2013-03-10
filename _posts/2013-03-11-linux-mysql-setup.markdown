---
layout: post
title: Правильная установка и настройка MySQL в Linux
category: MySQL
tags: centos, linux, mysql
author: gaHcep
year: 2013
month: 3
day: 11
published: true
summary: Ставим правильно сервер MySQL на CentOS - прописываем правила в iptables, меняем кодировку на UTF-8, корректно разбираемся с пользователями.
---

Эта статья, по замыслу, должна служить пошаговым руководством по корректной настройки MySQL сервера в Linux в общем и в CentOS в частности, начиная от подготовки системы и заканчивая настройкой прав пользователей.   
В этот раз текста будет минимум - только команды. 

Погнали.
 
# Установка MySQL

Обновляем систему

> `sudo yum update`

Проверяем, установлен ли MySQL сервер

> `mysql`

Если установлен, шаги по установке можете пропустить, хотя ознакомиться я все же советую с ними.

Существуют следующие основные пакеты связанные с mysql:

 * **mysql** - клиент mysql
 * **mysql-server** - сервер mysql
 * **mysql-devel** - для разработки и подключения библиотек и хидеров mysql
 * **mysql-connector-java** - JDBC коннектор (используется, например, в EJBCA)

Ставим:

> `sudo yum install mysql mysql-server mysql-devel mysql-connector-java`

Теперь надо установить сервер mysql на запуск в определенные **runlevel**'ы (2, 3 и 5):

> `chkconfig --level 235 mysqld on`

Если кто забыл соответствие цифрового значения **runlevel**'а символьному:

> `cat /etc/inittab`

поможет вспомнить.

Стартуем демон **mysql**:

> `sudo service mysqld start`

Получаем:

    ...  
    PLEASE REMEMBER TO SET A PASSWORD FOR THE MySQL root USER !  
    To do so, start the server, then issue the following commands:  

    /usr/bin/mysqladmin -u root password 'new-password'  
    /usr/bin/mysqladmin -u root -h localhost.localdomain password 'new-password'  

    Alternatively you can run:  
    /usr/bin/mysql_secure_installation  

    which will also give you the option of removing the test  
    databases and anonymous user created by default.  This is  
    strongly recommended for production servers.  
    ...

# Настройка сервера

Теперь пора настроить сервер. Начнем с пользователей.

Вот состояние таблицы **user** до начала действий с ней:

> mysql -u root  
  **_> use mysql_**  
  **_> select host,user from user;_**

>       +-----------------------+------+  
      | host                  | user |  
      +-----------------------+------+  
      | 127.0.0.1             | root |  
      | localhost             |      |  
      | localhost             | root |  
      | localhost.localdomain |      |  
      | localhost.localdomain | root |  
      +-----------------------+------+  
 > 5 rows in set (0.00 sec)  
  **_> quit_**

Как видете, безопасность на уровне плинтуса. Хорошо хоть, что анонимного пользователя нет.

Для настройки базовых вещей в сервере, запустим настройку сервера через **mysql\_secure\_installation**. На время этой установки, пароль будет **security**. Ваш же пароль, как понимаете, должен отличаться.

> `/usr/bin/mysql_secure_installation`

Запустится скрипт, с запросами на то или иное действие. Вот ответы:

1. **Skip root password for root**  
Мы еще не устанавливали пароль для root, поэтому при запуске скрипта и запросе пароля для `root`, просто нажмите `Enter`.

2. **Install new password for root: security**  
А вот тут можно установить пароль для `root`
  
3. **Do remove an anonymous user**  
На вопрос о том, удалить ли анонимного пользователя, отвечаем да

4. **Do not disallow remote connections**  
Не запрещаем коннект к нашему северу с удаленных серверов (если, конечно, эта опция вам нужна, в другом случае, запретите ее)

5. **Do remove a test database**  
Тестовая база нам не нужна - удаляйте ее

6. **Do reload the privileges**  
Перегрузим привилегии для их активации

Теперь для всех `root` пользователей установлен пароль.

Если в ходе этой конфигурации вы не установили пароль для `root`, можете сделать это так:

> `SET PASSWORD FOR 'root'@'localhost' = PASSWORD('security');`  
  `SET PASSWORD FOR 'root'@'localhost.localdomain' = PASSWORD('security');`  
  `SET PASSWORD FOR 'root'@'127.0.0.1' = PASSWORD('security');`  

или

> `UPDATE mysql.user SET Password = PASSWORD('security') WHERE user = 'root';`


Если же вы не запускали конфигурацию через `mysql_secure_installation` или не хотите этого делать по каким-то другим причинам, следующие команды удалят **any** пользователей:

> `DROP USER ''@'localhost';`  
> `DROP USER ''@'localhost.localdomain';`  

Также, пароль для IPv6 localhost (**@::1**) можно установить таким образом:

> `SET PASSWORD FOR 'root'@'::1' = PASSWORD(<password>);`

Близится финал нашего действия. Осталось две вещи:

 * открыть порты для mysql:

 > `sudo iptables -I INPUT -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT`  
 > `sudo iptables -I OUTPUT -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT`

 * выставить кодировку UTF-8 по-умолчанию - файл **`/etc/my.cnf`**:

    [mysqld]  
    init_connect='SET collation_connection = utf8_unicode_ci'  
    character-set-server = utf8  
    collation-server = utf8_unicode_ci

    [client]  
    default-character-set = utf8  


Перегружаем сервис:

 > `sudo service mysqld restart`


На этом все.
