запускаю контейнер в фоновом режиме<br>
docker run -d -p 3000:80 -it ecwid/ops-test-task:20210311a<br><br>

Проверка<br>
 > curl -l http://localhost:3000
<br>

```
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
```

<br><br>

Захожу в контейнер и смотрю что слушает сеть<br>

> docker exec -it d11996feb966 bash<br>
> netstat -tlpen<br>

```
Active Internet connections (only servers) <br>
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name 
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      102        20901454   - 
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      0          20897220   43/nginx: master pr 
tcp        0      0 127.0.0.1:8082          0.0.0.0:*               LISTEN      0          20901611   22/java 
tcp6       0      0 :::80                   :::*                    LISTEN      0          20897221   43/nginx: master pr 
```
<br><br>

Проверяю лог nginx<br>

> less /var/log/nginx/error.log <br>

```
2021/03/22 03:00:06 [error] 46#46: *1 connect() failed (111: Connection refused) while connecting to upstream, client: 172.17.0.1, server: _, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:8000/", host: "localhost:3000"
```
<br><br>

Он ищет приложение на порту 8000, но по выводу netstat оно запущено на 8082, правлю конфиг:<br>
> vim /etc/nginx/sites-enabled/box.conf<br>

```
proxy_pass http://localhost:8082;
```
<br><br>

Перезапускаю nginx, открываю страницу в браузере, но она не работает. В логах Nginx ошибок нет, смотрю лог приложения:
> less /var/log/box.log
<br>

```
2021-03-22 03:05:44 ERROR box[nioEventLoopGroup-4-1] ktor.application: Unhandled: GET - / 
java.lang.OutOfMemoryError: Java heap space 
```
<br><br>

Java нехватает памяти, ищу как запускается приложение<br>
> ps axwwf | grep java <br>

```
java -Xmx50m -classpath
```

<br>
<br>
Надо поправить количество выделяемой памяти<br>

> vim /etc/init.d/box
<br>

```
JAVA_OPTS='-Xmx512m'
```

<br><br>
Перезапускаю,  открывается страничка, ввожу "code" но получаю ошибку <br>

![screen](https://github.com/Zergvl/sample/blob/master/1/screenshots/2.JPG)

<br>
Смотрю лог приложения<br>
 > less /var/log/box.log
 <br>

```
2021-03-22 03:08:47 WARN box[nioEventLoopGroup-4-4] ktor.application: Cannot get email from the database, see the response for details
```

<br><br>

Не может прочитать адрес из БД. Помню что на 5432 висит postgresql, смотрю лог <br>

 > less /var/log/postgresql/postgresql-12-main.log <br>

```
2021-03-22 03:08:47.045 UTC [9861] box@box FATAL:  password authentication failed for user "box"
2021-03-22 03:08:47.045 UTC [9861] box@box DETAIL:  Role "box" does not exist. 
```

<br><br>

Нет пользователя. Перед созданием правлю pg_hba.conf для доступа к СУБД. <br>

> vim /etc/postgresql/12/main/pg_hba.conf

 <br>

```
local   all             postgres                                trust
```

 <br>
 <br>
 
> service postgresql reload
 <br>
 
 Создаю пользователся и БД (box@box) с паролем из /etc/box.properties (конфиг указан в строке запуска приложения "ps" )<br>
 
 > psql -U postgres

```
postgres=# create role box with login password 'iwwIEIeEiEDDecIEeIwC';
CREATE ROLE
postgres=# create database box;
CREATE DATABASE
postgres=# alter database box owner to box ;
ALTER DATABASE
```

 <br> <br>
 
импорт "схемы" найденной в папке с приложением.  <br>
 > /opt/box# psql -U postgres box < schema.sql
<br><br>

Выдаю права на таблицу и добавляю в неё свой адрес: <br>

```
box=# alter table devops owner to box ;
ALTER TABLE
box=# insert into devops values ( 'zergnado@gmail.com' );
INSERT 0 1
```

<br><br>

Перезагружаю веб-страничку.<br>
![screen](https://github.com/Zergvl/sample/blob/master/1/screenshots/1.JPG)
<br><br>


Костыльный Dockerfile <br>

```
FROM ecwid/ops-test-task:20210311a
RUN  sed -i -r "s/8000/8082/g" /etc/nginx/sites-enabled/box.conf &&  sed -i -r "s/-Xmx50m/-Xmx512m/g" /etc/init.d/box &&  sed -i -r "s/peer/trust/g" /etc/postgresql/12/main/pg_hba.conf && service postgresql stop && service postgresql start
COPY dump /opt/box/
RUN service postgresql start &&  /usr/bin/psql -U postgres < /opt/box/dump

EXPOSE 80/tcp
CMD ["/startup"]
```

содержимое dump <br>

```
--
-- PostgreSQL database cluster dump
--

CREATE ROLE box;
ALTER ROLE box WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN NOREPLICATION NOBYPASSRLS PASSWORD 'md5b2b8a3ca544e0a6d12c3300cf342ac0d';

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: box; Type: DATABASE; Schema: -; Owner: box
--

CREATE DATABASE box WITH TEMPLATE = template0 ENCODING = 'UTF8' LC_COLLATE = 'C.UTF-8' LC_CTYPE = 'C.UTF-8';


ALTER DATABASE box OWNER TO box;

\connect box

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;
SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: devops; Type: TABLE; Schema: public; Owner: box
--

CREATE TABLE public.devops (
    email character varying(100)
);


ALTER TABLE public.devops OWNER TO box;

--
-- Data for Name: devops; Type: TABLE DATA; Schema: public; Owner: box
--

COPY public.devops (email) FROM stdin;
somemail@mail.com
\.


--
-- PostgreSQL database cluster dump complete
--
```