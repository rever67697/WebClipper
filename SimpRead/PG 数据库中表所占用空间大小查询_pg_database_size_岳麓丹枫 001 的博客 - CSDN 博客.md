> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/yueludanfeng/article/details/86487585?utm_source=pocket_saves)

*   查看所有表所占磁盘空间大小

```
select sum(t.size) from (
SELECT table_schema || '.' || table_name AS table_full_name, pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')AS size
FROM information_schema.tables
ORDER BY
pg_total_relation_size('"' || table_schema || '"."' || table_name || '"') DESC
) t
```

*   查看每个表所占用磁盘空间大小

```
SELECT table_schema || '.' || table_name AS table_full_name, pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')AS size
FROM information_schema.tables
ORDER BY
pg_total_relation_size('"' || table_schema || '"."' || table_name || '"') DESC
```

*   查看数据库大小

```
playboy=> \l                       //\加上字母l,相当于mysql的，mysql> show databases;  
        List of databases  
   Name    |  Owner   | Encoding  
-----------+----------+----------  
 playboy   | postgres | UTF8  
 postgres  | postgres | UTF8  
 template0 | postgres | UTF8  
 template1 | postgres | UTF8  
  
playboy=> select pg_database_size('playboy');    //查看playboy数据库的大小  
 pg_database_size  
------------------  
(1 row)  
  
playboy=> select pg_database.datname, pg_database_size(pg_database.datname) AS size from pg_database;    //查看所有数据库的大小  
  datname  |  size  
-----------+---------  
 postgres  | 3621512  
 playboy   | 3637896  
 template1 | 3563524  
 template0 | 3563524  
(4 rows)  
  
playboy=> select pg_size_pretty(pg_database_size('playboy'));      //以KB，MB，GB的方式来查看数据库大小  
 pg_size_pretty  
----------------  
 3553 kB  
(1 row)
```

*   查看表大小

```
playboy=> \d test;                 //相当于mysql的，mysql> desc test;  
            Table "public.test"  
 Column |         Type          | Modifiers  
--------+-----------------------+-----------  
 id     | integer               | not null  
 name   | character varying(32) |  
Indexes: "playboy_id_pk" PRIMARY KEY, btree (id)  
  
playboy=> select pg_relation_size('test');   //查看表大小  
 pg_relation_size  
------------------  
(1 row)  
  
playboy=> select pg_size_pretty(pg_relation_size('test'));   //以KB，MB，GB的方式来查看表大小  
 pg_size_pretty  
----------------  
 0 bytes  
(1 row)  
  
playboy=> select pg_size_pretty(pg_total_relation_size('test'));   //查看表的总大小，包括索引大小  
 pg_size_pretty  
----------------  
 8192 bytes  
(1 row)
```

*   查看所有所占磁盘空间大小

```
playboy=> \di                      //相当于mysql的，mysql> show index from test;  
                List of relations  
 Schema |     Name      | Type  |  Owner  | Table  
--------+---------------+-------+---------+-------  
 public | playboy_id_pk | index | playboy | test  
(1 row)  
  
playboy=> select pg_size_pretty(pg_relation_size('playboy_id_pk'));    //查看索大小  
 pg_size_pretty  
----------------  
 8192 bytes  
(1 row)
```

*   查看表空间大小

```
playboy=> select spcname from pg_tablespace;         //查看所有表空间  
  spcname  
------------  
 pg_default  
 pg_global  
(2 rows)  
  
playboy=> select pg_size_pretty(pg_tablespace_size('pg_default'));   //查看表空间大小  
 pg_size_pretty  
----------------  
 14 MB  
(1 row)
```