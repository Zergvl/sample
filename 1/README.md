запускаю контейнер в фоновом режиме<br>
docker run -d -p 3000:80 -it ecwid/ops-test-task:20210311a<br><br>

Проверка<br>
curl -l http://localhost:3000<br>

<code>
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
</code>
<br><br>

Захожу в контейнер и смотрю что слушает сеть<br>

> docker exec -it d11996feb966 bash<br>
> netstat -tlpen<br>

<code>
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      102        20901454   -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      0          20897220   43/nginx: master pr
tcp        0      0 127.0.0.1:8082          0.0.0.0:*               LISTEN      0          20901611   22/java
tcp6       0      0 :::80                   :::*                    LISTEN      0          20897221   43/nginx: master pr
</code>
<br><br>

Проверяю лог nginx<br>

> less /var/log/nginx/error.log <br>
<code>
2021/03/22 03:00:06 [error] 46#46: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 172.17.0.1, server: _, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:8000/", host: "localhost:3000"
</code>
<br><br>

Он ищет приложение на порту 8000, но по выводу netstat оно запущено на 8082, правлю конфиг:<br>
> vim /etc/nginx/sites-enabled/box.conf<br>
<code>
proxy_pass http://localhost:8082;
</code>
<br><br>

Перезапускаю nginx, открываю страницу в браузере, но она не работает. В логах Nginx ошибок нет, смотрю лог приложения:
> less /var/log/box.log
</code>
2021-03-22 03:05:44 ERROR box[nioEventLoopGroup-4-1] ktor.application: Unhandled: GET - /
java.lang.OutOfMemoryError: Java heap space
</code>
<br><br>

Java нехватает памяти, ищу как запускается приложение<br>
> ps axwwf | grep java<br>
<code>
java -Xmx50m -classpath
</code>
<br>
Надо поправить количество выделяемой памяти<br>

> vim /etc/init.d/box
<br>
<code>
JAVA_OPTS='-Xmx512m'
</code>
<br><br>
Перезапускаю,  открывается страничка, ввожу "code" но получаю ошибку <br>

![screen](https://github.com/Zergvl/sample/blob/master/1/screenshots/2.JPG)

<br>
Смотрю лог приложения<br>
 > less /var/log/box.log
 <br>
<code>
2021-03-22 03:51:31 WARN box[nioEventLoopGroup-4-4] ktor.application: Cannot get email from the database, see the response for details
</code>
<br><br>

Не может прочитать адрес из БД. Помню что на 5432 висит postgresql, смотрю лог<br>
<code>
less /var/log/postgresql/postgresql-12-main.log
2021-03-22 03:08:47.045 UTC [9861] box@box FATAL:  password authentication failed for user "box"
2021-03-22 03:08:47.045 UTC [9861] box@box DETAIL:  Role "box" does not exist.
</code>
<br><br>
Нет пользователя. Перед созданием правлю pg_hba.conf для доступа к СУБД. <br>

 > vim /etc/postgresql/12/main/pg_hba.conf
 <br>
 <code>
local   all             postgres                                trust
</code>
 <br>
> service postgresql reload
 <br>
 
 Создаю пользователся и БД (box@box) с паролем из /etc/box.properties (конфиг указан в строке запуска приложения "ps" )<br>
 
 > psql -U postgres
 <code>
postgres=# create role box with login password 'iwwIEIeEiEDDecIEeIwC';
CREATE ROLE
postgres=# create database box;
CREATE DATABASE
postgres=# alter database box owner to box ;
ALTER DATABASE
</code>
 <br> <br>
импорт "схемы" найденной в папке с приложением.  <br>
 > /opt/box# psql -U postgres box < schema.sql
<br><br>

Выдаю права на таблицу и добавляю в неё свой адрес: <br>
 <code>
box=# alter table devops owner to box ;
ALTER TABLE
box=# insert into devops values ( 'zergnado@gmail.com' );
INSERT 0 1
</code>
<br><br>

Перезагружаю веб-страничку.<br>
![screen](https://github.com/Zergvl/sample/blob/master/1/screenshots/1.JPG)
<br><br>
