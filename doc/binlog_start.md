# 初始binlog

binlog需要开启`binlog_format=row`，建议是行模式.

实验结论：

| 语句类型     | binlog记录情况     | 备注                                                            |
|------------|-----------------|-----------------------------------------------------------------|
| 建表         | 记录             | 记录完整建表语句                |
| 查询         | 不记录             | 查询包括select/show等， 不管是否是事务中，都不记录                |
| 新增         | 记录完整的新增字段 | 默认字段/自增字段，在insert语句中不包含的，在binlog中也会提现出来 |
| 修改 影响0行 | 不记录             |                                                                 |
| 修改 影响n行 | 记录n行            | 包含n行的修改前的字段值，以及对应的修改后的字段值                |
| 删除         | 同修改             |                                                                 |

其中查询不被记录，是在mysql的[官方文档](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)中显式地说明了的：

> **The binary log is not used for statements such as SELECT or SHOW that do not modify data**. To log all statements (for example, to identify a problem query), use the general query log. See Section 5.4.3, “The General Query Log”.

思考题：

> 误删除表记录，可以通过binlog给找回来(将binlog中对应的delete转换成insert即可)，如果手抖误drop删除表，能不能通过binlog找回来呢？（答案在文末）

## 实验查询

### 建表

```bash
🕙[2020-09-10 17:17:00.032] ❯ mysql -uroot -proot -h 127.0.0.1 -P 3311
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.29-log MySQL Community Server (GPL)
mysql> use bjca;
Database changed
mysql> create table card
    -> (
    ->     id          bigint auto_increment primary key comment '自增ID',
    ->     created     datetime default current_timestamp comment '创建时间',
    ->     updated     datetime on update current_timestamp comment '更新时间',
    ->     num     varchar(36) not null comment '卡号',
    ->     code   varchar(36) not null comment '卡密',
    ->     state int not null default 0 comment '状态。 0:未用 1:预占 2:已用'
    -> ) engine = innodb
    ->   default charset = utf8mb4 comment '卡表';
Query OK, 0 rows affected (0.03 sec)

mysql> show binary logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000005 |      4191 |
| mysql-bin.000006 |      2872 |
+------------------+-----------+
2 rows in set (0.00 sec)
```

```bash
🕙[2020-09-10 17:17:01.452] ❯ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
d0902ae1e08b        mysql:5.7.29        "docker-entrypoint.s…"   3 months ago        Up 3 hours          33060/tcp, 0.0.0.0:3311->3306/tcp   otter_mysqla_1

mysql-cluster/otter on  master [!]
🕙[2020-09-10 17:17:23.537] ❯ docker exec -it otter_mysqla_1 bash
root@d0902ae1e08b:/# cd /var/lib/mysql
root@d0902ae1e08b:/var/lib/mysql# mysqlbinlog --database bjca --base64-output=decode-rows --verbose mysql-bin.000006 --start-datetime "2020-09-10 09:17:00"
create table card
(
    id          bigint auto_increment primary key comment '自增ID',
    created     datetime default current_timestamp comment '创建时间',
    updated     datetime on update current_timestamp comment '更新时间',
    num     varchar(36) not null comment '卡号',
    code   varchar(36) not null comment '卡密',
    state int not null default 0 comment '状态。 0:未用 1:预占 2:已用'
) engine = innodb
  default charset = utf8mb4 comment '卡表'
/*!*/;
DELIMITER ;
# End of log file
```

### 插入记录

新增1条，再查询

```bash
mysql> insert into card(num, code) values('100','aaa');
Query OK, 1 row affected (0.01 sec)

mysql> select * from card;
+----+---------------------+---------+-----+------+-------+
| id | created             | updated | num | code | state |
+----+---------------------+---------+-----+------+-------+
|  1 | 2020-09-10 09:26:54 | NULL    | 100 | aaa  |     0 |
+----+---------------------+---------+-----+------+-------+
1 row in set (0.00 sec)
```

看binlog：

```bash
root@d0902ae1e08b:/var/lib/mysql# mysqlbinlog --database bjca --base64-output=decode-rows --verbose mysql-bin.000006 --start-datetime "2020-09-10 09:26:44"
#200910  9:26:54 server id 10001  end_log_pos 3141 CRC32 0x814ae9e4 	Write_rows: table id 111 flags: STMT_END_F
### INSERT INTO `bjca`.`card`
### SET
###   @1=1
###   @2='2020-09-10 09:26:54'
###   @3=NULL
###   @4='100'
###   @5='aaa'
###   @6=0
# at 3141
#200910  9:26:54 server id 10001  end_log_pos 3172 CRC32 0x70069907 	Xid = 72
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
```

批量新增2条，再查询

```bash
mysql> insert into card(num, code) values('200','bbb'), ('300', 'ccc');
Query OK, 2 rows affected (0.01 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql> select * from card;
+----+---------------------+---------+-----+------+-------+
| id | created             | updated | num | code | state |
+----+---------------------+---------+-----+------+-------+
|  1 | 2020-09-10 09:26:54 | NULL    | 100 | aaa  |     0 |
|  3 | 2020-09-10 09:29:20 | NULL    | 200 | bbb  |     0 |
|  5 | 2020-09-10 09:29:20 | NULL    | 300 | ccc  |     0 |
+----+---------------------+---------+-----+------+-------+
3 rows in set (0.00 sec)
```

看binlog:

```bash
root@d0902ae1e08b:/var/lib/mysql# mysqlbinlog --database bjca --base64-output=decode-rows --verbose mysql-bin.000006 --start-datetime "2020-09-10 09:29:00"
BEGIN
/*!*/;
# at 3322
#200910  9:29:20 server id 10001  end_log_pos 3380 CRC32 0x758e34f3 	Table_map: `bjca`.`card` mapped to number 111
# at 3380
#200910  9:29:20 server id 10001  end_log_pos 3467 CRC32 0x21705f6c 	Write_rows: table id 111 flags: STMT_END_F
### INSERT INTO `bjca`.`card`
### SET
###   @1=3
###   @2='2020-09-10 09:29:20'
###   @3=NULL
###   @4='200'
###   @5='bbb'
###   @6=0
### INSERT INTO `bjca`.`card`
### SET
###   @1=5
###   @2='2020-09-10 09:29:20'
###   @3=NULL
###   @4='300'
###   @5='ccc'
###   @6=0
# at 3467
#200910  9:29:20 server id 10001  end_log_pos 3498 CRC32 0xbae087e0 	Xid = 74
COMMIT/*!*/;
```

### 删除记录

直接全部删除：

```bash
mysql> delete  from card;
Query OK, 3 rows affected (0.01 sec)
```

看binlog:

```bash
### DELETE FROM `bjca`.`card`
### WHERE
###   @1=1
###   @2='2020-09-10 09:26:54'
###   @3=NULL
###   @4='100'
###   @5='aaa'
###   @6=0
### DELETE FROM `bjca`.`card`
### WHERE
###   @1=3
###   @2='2020-09-10 09:29:20'
###   @3=NULL
###   @4='200'
###   @5='bbb'
###   @6=0
### DELETE FROM `bjca`.`card`
### WHERE
###   @1=5
###   @2='2020-09-10 09:29:20'
###   @3=NULL
###   @4='300'
###   @5='ccc'
###   @6=0
```

### 删除表

看看危险的动作，直接drop删除表，binlog会怎么应对

```bash
mysql> insert into card(num, code) values('200','bbb'), ('300', 'ccc');
Query OK, 2 rows affected (0.00 sec)
Records: 2  Duplicates: 0  Warnings: 0

mysql>
mysql>
mysql>
mysql> drop table card;
Query OK, 0 rows affected (0.01 sec)
```

binlog结果

```bash
root@d0902ae1e08b:/var/lib/mysql# mysqlbinlog --database bjca --base64-output=decode-rows --verbose mysql-bin.000006 --start-datetime "2020-09-10 09:34:04"
DROP TABLE `card` /* generated by server */
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
```

结果是，binlog只是记录了drop表语句，原有的记录，并没有记录下来。
