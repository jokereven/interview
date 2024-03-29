# Binlog

## 1.1. 概念

MySQL中一般有一下几种日志

|         AID         |                             Name                             |
| :-----------------: | :----------------------------------------------------------: |
|      日志类型       |                        写入日志的信息                        |
|      错误日志       |          记录在启动,运行或者停止mysqld时遇到的问题           |
|    通用查询日志     |               记录建立的客户端连接和执行的语句               |
|     二进制日志      |                      记录更改数据的语句                      |
|      中继日志       |                 从复制主服务器接收的数据更改                 |
|     慢查询日志      | 记录所有执行时间超过`long_query_time` 秒的所有查询或者不使用索引的查询 |
| DDL日志(元数据日志) |                   元数据操作由DDL语句执行                    |

MySQL 的二进制日志 binlog 可以说是 MySQL 最重要的日志，它记录了所有的 DDL 和 DML 语句（除了数据查询语句select、show等），以事件形式记录，还包含语句所执行的消耗的时间，MySQL的二进制日志是事务安全型的。binlog 的主要目的是复制和恢复。

binlog是MySQL server层维护的一种二进制日志,与innodb引擎里的redo/undo log 是完全不同的日志;其主要要用来记录对mysql数据更新或潜在发生更新的SQL语句,并以"事务"的形式保存在磁盘中；

作用主要有：

- 复制:MySQL Replication在Master端开启binlog，Master把它的二进制日志传递给slaves并回放来达到master-slave数据一致的目的
- 数据恢复: 通过mysqlbinlog工具恢复数据
- 增量备份

##  1.2. binlog管理

开启binlogmy.cnf配置中设置：log_bin="存放binlog路径目录"

> 一般来说开启binlog日志大概会有1%的性能损耗。

binlog信息查询binlog开启后，可以在配置文件中查看其位置信息，也可以在myslq命令行中查看：

```sql
show variables like '%log_bin%';
+---------------------------------+-------------------------------------+
| Variable_name                   | Value                               |
+---------------------------------+-------------------------------------+
| log_bin                         | ON                                  |
| log_bin_basename                | /var/lib/mysql/3306/mysql-bin       |
| log_bin_index                   | /var/lib/mysql/3306/mysql-bin.index |
| log_bin_trust_function_creators | OFF                                 |
| log_bin_use_v1_row_events       | OFF                                 |
| sql_log_bin                     | ON                                  |
+---------------------------------+-------------------------------------+
Copy
```

binlog文件开启binlog后，会在数据目录（默认）生产host-bin.n（具体binlog信息）文件及host-bin.index索引文件（记录binlog文件列表）。当binlog日志写满(binlog大小max_binlog_size，默认1G),或者数据库重启才会生产新文件，但是也可通过手工进行切换让其重新生成新的文件（flush logs）；另外，如果正使用大的事务，由于一个事务不能横跨两个文件，因此也可能在binlog文件未满的情况下刷新文件

```sql
mysql> show binary logs; //查看binlog文件列表,
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       177 |
| mysql-bin.000002 |       177 |
| mysql-bin.000003 |  10343266 |
| mysql-bin.000004 |  10485660 |
| mysql-bin.000005 |     53177 |
| mysql-bin.000006 |      2177 |
| mysql-bin.000007 |      1383 |
+------------------+-----------+
Copy
```

查看binlog的状态：show master status可查看当前二进制日志文件的状态信息，显示正在写入的二进制文件，及当前position

```sql
mysql> show master status;
 +------------------+----------+--------------+------------------+-------------------+
 | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
 +------------------+----------+--------------+------------------+-------------------+
 | mysql-bin.000007 |      120 |              |                  |                   |
 +------------------+----------+--------------+------------------+-------------------+
Copy
```

reset master 清空binlog日志文件

## 1.3. binlog内容

默认情况下binlog日志是二进制格式，无法直接查看。可使用两种方式进行查看：

1. mysqlbinlog: /usr/bin/mysqlbinlog mysql-bin.000007

   - mysqlbinlog是mysql官方提供的一个binlog查看工具，
   - 也可使用–read-from-remote-server从远程服务器读取二进制日志，
   - 还可使用--start-position --stop-position、--start-time= --stop-time精确解析binlog日志

   ```
    截取位置1190-1352 binlog如下：
            ***************************************************************************************
            # at 1190   //事件的起点
            #171223 21:56:26 server id 123  end_log_pos 1190 CRC32 0xf75c94a7     Intvar
            SET INSERT_ID=2/*!*/;
            #171223 21:56:26 server id 123  end_log_pos 1352 CRC32 0xefa42fea     Query    thread_id=4    exec_time=0    error_code=0
            SET TIMESTAMP=1514123786/*!*/;              //开始事务的时间起点 (每个at即为一个event)
            insert into tb_person  set name="name__2", address="beijing", sex="man", other="nothing"  //sql语句
            /*!*/;
            # at 1352
            #171223 21:56:26 server id 123  end_log_pos 1383 CRC32 0x72c565d3     Xid = 5 //执行时间，及位置戳，Xid:事件指示提交的XA事务
            ***************************************************************************************
   Copy
   ```

2. 直命令行解析

   ```sql
    SHOW BINLOG EVENTS
         [IN 'log_name'] //要查询的binlog文件名
         [FROM pos]  
         [LIMIT [offset,] row_count]
   Copy
   ```

   1190-135如下：mysql> show binlog events in 'mysql-bin.000007' from 1190 limit 2\G

   ```sql
       *************************** 13. row ***************************
          Log_name: mysql-bin.000007
               Pos: 1190
        Event_type: Query  //事件类型
         Server_id: 123
       End_log_pos: 1352   //结束pose点，下个事件的起点
              Info: use `test`; insert into tb_person  set name="name__2", address="beijing", sex="man", other="nothing"
       *************************** 14. row ***************************
          Log_name: mysql-bin.000007
               Pos: 1352
        Event_type: Xid
         Server_id: 123
       End_log_pos: 1383
              Info: COMMIT /* xid=51 */
   ```

## 1.4. binlog格式

Mysql binlog日志有`ROW`,`Statement`,`MiXED`三种格式；

可通过my.cnf配置文件及 ==set global binlog_format='ROW/STATEMENT/MIXED'== 进行修改，

命令行 ==show variables like 'binlog_format'== 命令查看binglog格式；

> 在 MySQL 5.7.7 之前，默认的格式是 STATEMENT，在 MySQL 5.7.7 及更高版本中，默认值是 ROW。日志格式通过 binlog-format 指定，如 binlog-format=STATEMENT、binlog-format=ROW、binlog-format=MIXED。

- Row level:
  - 概念:
    - 仅保存记录被修改细节，不记录sql语句上下文相关信息
  - 优点：能非常清晰的记录下每行数据的修改细节，不需要记录上下文相关信息，因此不会发生某些特定情况下的procedure、function、及trigger的调用触发无法被正确复制的问题，任何情况都可以被复制，且能加快从库重放日志的效率，保证从库数据的一致性
  - 缺点:
    - 由于所有的执行的语句在日志中都将以每行记录的修改细节来记录，因此，可能会产生大量的日志内容，干扰内容也较多；比如一条update语句，如修改多条记录，则binlog中每一条修改都会有记录，这样造成binlog日志量会很大，特别是当执行alter table之类的语句的时候，由于表结构修改，每条记录都发生改变，那么该表每一条记录都会记录到日志中，实际等于重建了表。
  - tip:
    - row模式生成的sql编码需要解码，不能用常规的办法去生成，需要加上相应的参数(--base64-output=decode-rows -v)才能显示出sql语句; - 新版本binlog默认为ROW level，且5.6新增了一个参数：binlog_row_image；把binlog_row_image设置为minimal以后，binlog记录的就只是影响的列，大大减少了日志内容
- Statement level:
  - 概念:
    - 每一条会修改数据的sql都会记录在binlog中
  - 优点：只需要记录执行语句的细节和上下文环境，避免了记录每一行的变化，在一些修改记录较多的情况下相比ROW level能大大减少binlog日志量，节约IO，提高性能；还可以用于实时的还原；同时主从版本可以不一样，从服务器版本可以比主服务器版本高
  - 缺点：
    - 为了保证sql语句能在slave上正确执行，必须记录上下文信息，以保证所有语句能在slave得到和在master端执行时候相同的结果；另外，主从复制时，存在部分函数（如sleep）及存储过程在slave上会出现与master结果不一致的情况，而相比Row level记录每一行的变化细节，绝不会发生这种不一致的情况
- Mixedlevel level:
  - 操作:
    - 以上两种level的混合使用经过前面的对比，可以发现ROW level和statement level各有优势，如能根据sql语句取舍可能会有更好地性能和效果；Mixed level便是以上两种leve的结合。不过，新版本的MySQL对row level模式也被做了优化，并不是所有的修改都会以row level来记录，像遇到表结构变更的时候就会以statement模式来记录，如果sql语句确实就是update或者delete等修改数据的语句，那么还是会记录所有行的变更；因此，现在一般使用row level即可。
- 选取规则如果是采用 INSERT，UPDATE，DELETE 直接操作表的情况，则日志格式根据 binlog_format 的设定而记录
- 如果是采用 GRANT，REVOKE，SET PASSWORD 等管理语句来做的话，那么无论如何都采用statement模式记录

## 1.5. 复制

复制是mysql最重要的功能之一，mysql集群的高可用、负载均衡和读写分离都是基于复制来实现的；从5.6开始复制有两种实现方式，基于binlog和基于GTID（全局事务标示符）；本文接下来将介绍基于binlog的一主一从复制；其复制的基本过程如下：

1. Master将数据改变记录到二进制日志(binary log)中
2. Slave上面的IO进程连接上Master，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容
3. Master接收到来自Slave的IO进程的请求后，负责复制的IO进程会根据请求信息读取日志指定位置之后的日志信息，返回给Slave的IO进程。 返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到Master端的bin-log文件的名称以及bin-log的位置
4. Slave的IO进程接收到信息后，将接收到的日志内容依次添加到Slave端的relay-log文件的最末端，并将读取到的Master端的 bin-log的 文件名和位置记录到master-info文件中，以便在下一次读取的时候能够清楚的告诉Master从某个bin-log的哪个位置开始往后的日志内容
5. Slave的Sql进程检测到relay-log中新增加了内容后，会马上解析relay-log的内容成为在Master端真实执行时候的那些可执行的内容，并在自身执行

接下来使用实例演示基于binlog的主从复制：

```
a.配置master
        主要包括设置复制账号，并授予REPLICATION SLAVE权限，具体信息会存储在于master.info文件中，及开启binlog；
        mysql> CREATE USER 'test'@'%' IDENTIFIED BY '123456';
        mysql> GRANT REPLICATION SLAVE ON *.* TO 'test'@'%';
        mysql> show variables like "log_bin";
            +---------------+-------+
            | Variable_name | Value |
            +---------------+-------+
            | log_bin       | ON    |
            +---------------+-------+
        查看master当前binlogmysql状态：mysql> show master status;
            +------------------+----------+--------------+------------------+-------------------+
            | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
            +------------------+----------+--------------+------------------+-------------------+
            | mysql-bin.000003 |      120 |              |                  |                   |
            +------------------+----------+--------------+------------------+-------------------+
        建表插入数据：
            CREATE TABLE `tb_person` (
               `id` int(11) NOT NULL AUTO_INCREMENT,
               `name` varchar(36) NOT NULL,                           
               `address` varchar(36) NOT NULL DEFAULT '',    
               `sex` varchar(12) NOT NULL DEFAULT 'Man' ,
               `other` varchar(256) NOT NULL ,
               PRIMARY KEY (`id`)
             ) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8;

             insert into tb_person  set name="name1", address="beijing", sex="man", other="nothing";
             insert into tb_person  set name="name2", address="beijing", sex="man", other="nothing";
             insert into tb_person  set name="name3", address="beijing", sex="man", other="nothing";
             insert into tb_person  set name="name4", address="beijing", sex="man", other="nothing";
    b.配置slave
        Slave的配置类似master，需额外设置relay_log参数，slave没有必要开启二进制日志，如果slave为其它slave的master，须设置bin_log
    c.连接master
        mysql> CHANGE MASTER TO
           MASTER_HOST='10.108.111.14',
           MASTER_USER='test',
           MASTER_PASSWORD='123456',
           MASTER_LOG_FILE='mysql-bin.000003',
           MASTER_LOG_POS=120;
    d.show slave status;
        mysql> show slave status\G
        *************************** 1. row ***************************
                       Slave_IO_State:   ---------------------------- slave io状态，表示还未启动
                          Master_Host: 10.108.111.14  
                          Master_User: test  
                          Master_Port: 20126  
                        Connect_Retry: 60   ------------------------- master宕机或连接丢失从服务器线程重新尝试连接主服务器之前睡眠时间
                      Master_Log_File: mysql-bin.000003  ------------ 当前读取master binlog文件
                  Read_Master_Log_Pos: 120  ------------------------- slave读取master binlog文件位置
                       Relay_Log_File: relay-bin.000001  ------------ 回放binlog
                        Relay_Log_Pos: 4   -------------------------- 回放relay log位置
                Relay_Master_Log_File: mysql-bin.000003  ------------ 回放log对应maser binlog文件
                     Slave_IO_Running: No
                    Slave_SQL_Running: No
                  Exec_Master_Log_Pos: 0  --------------------------- 相对于master从库的sql线程执行到的位置
                Seconds_Behind_Master: NULL
        Slave_IO_State, Slave_IO_Running, 和Slave_SQL_Running为NO说明slave还没有开始复制过程。
    e.启动复制
        start slave
    f.再次观察slave状态
        mysql> show slave status\G
        *************************** 1. row ***************************
                       Slave_IO_State: Waiting for master to send event -- 等待master新的event
                          Master_Host: 10.108.111.14
                          Master_User: test
                          Master_Port: 20126
                        Connect_Retry: 60
                      Master_Log_File: mysql-bin.000003
                  Read_Master_Log_Pos: 3469  ---------------------------- 3469  等于Exec_Master_Log_Pos，已完成回放
                       Relay_Log_File: relay-bin.000002                    ||
                        Relay_Log_Pos: 1423                                ||
                Relay_Master_Log_File: mysql-bin.000003                    ||
                     Slave_IO_Running: Yes                                 ||
                    Slave_SQL_Running: Yes                                 ||
                  Exec_Master_Log_Pos: 3469  -----------------------------3469  等于slave读取master binlog位置，已完成回放
                Seconds_Behind_Master: 0
        可看到slave的I/O和SQL线程都已经开始运行，而且Seconds_Behind_Master=0。Relay_Log_Pos增加，意味着一些事件被获取并执行了。

        最后看下如何正确判断SLAVE的延迟情况，判定slave是否追上master的binlog：
        1、首先看 Relay_Master_Log_File 和 Maser_Log_File 是否有差异；
        2、如果Relay_Master_Log_File 和 Master_Log_File 是一样的话，再来看Exec_Master_Log_Pos 和 Read_Master_Log_Pos 的差异，对比SQL线程比IO线程慢了多少个binlog事件；
        3、如果Relay_Master_Log_File 和 Master_Log_File 不一样，那说明延迟可能较大，需要从MASTER上取得binlog status，判断当前的binlog和MASTER上的差距；
        4、如果以上都不能发现问题，可使用pt_heartbeat工具来监控主备复制的延迟。

    g.查询slave数据，主从一致
        mysql> select * from tb_person;
            +----+-------+---------+-----+---------+
            | id | name  | address | sex | other   |
            +----+-------+---------+-----+---------+
            |  5 | name4 | beijing | man | nothing |
            |  6 | name2 | beijing | man | nothing |
            |  7 | name1 | beijing | man | nothing |
            |  8 | name3 | beijing | man | nothing |
            +----+-------+---------+-----+---------+
关于mysql复制的内容还有很多，比如不同的同步方式、复制格式情况下有什么区别，有什么特点，应该在什么情况下使用....这里不再一一介绍。
```

## 1.6. 恢复

```
恢复是binlog的两大主要作用之一，接下来通过实例演示如何利用binlog恢复数据：

    a.首先，看下当前binlog位置
        mysql> show master status;
        +------------------+----------+--------------+------------------+-------------------+
        | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
        +------------------+----------+--------------+------------------+-------------------+
        | mysql-bin.000008 |     1847 |              |                  |                   |
        +------------------+----------+--------------+------------------+-------------------+
    b.向表tb_person中插入两条记录：
        insert into tb_person  set name="person_1", address="beijing", sex="man", other="test-1";
        insert into tb_person  set name="person_2", address="beijing", sex="man", other="test-2";
    c.记录当前binlog位置：
        mysql> show master status;
        +------------------+----------+--------------+------------------+-------------------+
        | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
        +------------------+----------+--------------+------------------+-------------------+
        | mysql-bin.000008 |     2585 |              |                  |                   |
        +------------------+----------+--------------+------------------+-------------------+
    d.查询数据 
        mysql> select *  from tb_person where name ="person_2" or name="person_1";
        +----+----------+---------+-----+--------+
        | id | name     | address | sex | other  |
        +----+----------+---------+-----+--------+
        |  6 | person_1 | beijing | man | test-1 |
        |  7 | person_2 | beijing | man | test-2 |
        +----+----------+---------+-----+--------+
    e.删除一条: delete from tb_person where name ="person_2";
        mysql> select *  from tb_person where name ="person_2" or name="person_1";
        +----+----------+---------+-----+--------+
        | id | name     | address | sex | other  |
        +----+----------+---------+-----+--------+
        |  6 | person_1 | beijing | man | test-1 |
        +----+----------+---------+-----+--------+
    f. binlog恢复（指定pos点恢复/部分恢复）
        mysqlbinlog   --start-position=1847  --stop-position=2585  mysql-bin.000008  > test.sql
        mysql> source /var/lib/mysql/3306/test.sql
    d.数据恢复完成 
        mysql> select *  from tb_person where name ="person_2" or name="person_1";
        +----+----------+---------+-----+--------+
        | id | name     | address | sex | other  |
        +----+----------+---------+-----+--------+
        |  6 | person_1 | beijing | man | test-1 |
        |  7 | person_2 | beijing | man | test-2 |
        +----+----------+---------+-----+--------+
    e.总结
        恢复，就是让mysql将保存在binlog日志中指定段落区间的sql语句逐个重新执行一次而已
```