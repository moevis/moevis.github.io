---
published: true
layout: post
categories: bug
autor: Moevis
---
感觉是有一些客户端没有正确调用 mysql_close 来退出，导致 too many connection 的问题。我想用命令行登进去都不行

```shell
$ mysql -u root -p
Enter password:
ERROR 1040 (HY000): Too many connections
```

后来我直接先关掉几个 mysql 客户端，然后再连进去就可以了。

连进去后首先先执行：

```sql
SHOW VARIABLES LIKE "max_connections";
```

这个显示的是最大连接数，默认是 151。

```
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
```

接着确认一下当前 process 的数量：

```sql
show processlist;
```

process 如下，总数也是 151，所以已经无法再连入新客户端了。

```
+-------+------+--------------+-----------+---------+------+----------+------------------+
| Id    | User | Host         | db        | Command | Time | State    | Info             |
+-------+------+--------------+-----------+---------+------+----------+------------------+
| 50540 | root | host:35772   | db_a      | Sleep   |   28 |          | NULL             |
| 50617 | root | host:52416   | db_b      | Sleep   |    0 |          | NULL             |
| 50618 | root | host:52418   | db_a      | Sleep   |   11 |          | NULL             |
| 50619 | root | host:52420   | db_b      | Sleep   |    0 |          | NULL             |
| 50620 | root | host:52422   | db_a      | Sleep   |    5 |          | NULL             |
| 50621 | root | host:52424   | NULL      | Sleep   |  262 |          | NULL             |
| 50622 | root | host:52426   | NULL      | Sleep   |  262 |          | NULL             |
```

我们现在希望找出那些没用的数据库连接，然后删除掉。其中 host 和 db_a, db_b 被我打码，可略过。而有的 db 为 NULL 表示那个连接是没有使用任何数据库的，所以可以安全删掉，停止对应进程的命令是 kill id。

processlist 实际上是存放在 information_schema 表中，所以可以用 sql 语句获取所有 db 为 NULL 的连接，并用 kill 语句结束进程：

```
select concat('kill ', ID, ';')select * from PROCESSLIST from PROCESSLIST where DB is null;
```

接着把输出的内容再粘贴回去运行即可。

还有一个步骤，就是把 process 上限提高：

```sql
set global max_connections = XXXX;
```
