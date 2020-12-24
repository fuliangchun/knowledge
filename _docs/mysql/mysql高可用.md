---
title: mysql高可用
category: mysql
order: 4
---



### Mysql高可用

#### 主从复制

mysql的高可用其实都是基于binlog来实现的

主从建立关系流程：

```sql
-- 主库创建用户
create user 'user_name'@'192.168.2.100' identified by 'iLtJokrjkhJSELO55Yz9';
grant insert,delete,update,select on  db_name.* to 'user_name'@'192.168.2.100';
-- 从库设置master的地址，以及使用的用户名和密码，以及BInlog开始的位置
CHANGE MASTER TO
  MASTER_HOST='192.168.2.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='repl123',
  MASTER_PORT=3306,
  master_log_file='mysql-bin.000004', 
  master_log_pos=194;
  -- 启动slave，查看主从关系是否正常建立
  start slave;
  show slave status\G; 
```


![](https://fuliangchun.github.io/knowledge/images/masterslave.png)

如图就是mysql的主从复制的流程图.主库写入binlog会有一个dump线程将binlog同步到从库。从库收到之后写入从库的relay log，relay log再起线程写入数据。

**在进行主从的时候推荐将binlog的模式设置为row，至少也要设置为mixed**因为在进行某些操作的时候可能会因为主从之间的不同，导致执行的结果不同，最终导致从库的数据和主库不一致。这个是非常致命的.row模式的缺点是会导致binlog日志的大小变化的非常快，但是row模式带来的好处方便**数据恢复**

问题：为什么主从之间的数据有可能会差别几个小时?
通过在备库中执行 show slave status，seconds_behind_master，用于表示当前备库延迟了多少秒,造成延迟的原因：

* 备库的机器性能比主库差  可能会有的考虑是说反正备库没有请求，备库采用比较差性能的机器，但其实可以看做是Binlog中更新操作也会有大量的读操作，对于Iops其实是和主库相同的
* 备库的压力一般来说比主库要大，备库需要执行主库过来的binlog，某些我们会进行读写分离，读请求到备库了，而且同时也需要执行主库的binlog，会导致从库压力要大些。而且如果是从库执行一些数据量比较大的读请求(用于导出数据)，内存页淘汰，导致利用率不高，从而影响性能**顺丰有一个导出功能，将数据请求到了备库**
* 主库中执行大事务，因为主库是在事务提交之后才写入binlog的，主库执行10分钟，备库就落后备库10分钟，这就天然的备库会落后主库。(大表的ddl也是一个延迟的原因)

常见的主从结构

- 一主多从 复制采用的是主推的模式，如果从库比较多的话，从库的数据库延迟比较大,且消耗主库的io
- 多级复制 一级一级的复制.
- 双主复制 双主互为主从，采用浮动ip技术(keepAlived)  
- 双主多级复制



这里要提一下双主，双主是互为主从，但是同一时刻都是只有一个接受用户请求的，在原来的A故障或者是需要切换的时候另外一台才会接受请求.图中即为双主的切换过程
![](https://fuliangchun.github.io/knowledge/images/masterslaveprocess.png)





### 分区表

分区表很多种类型，不过一般的话常用的range这种类型，一般是通过时间来进行区分

```sql
CREATE TABLE `t` (
  `ftime` datetime NOT NULL,
  `c` int(11) DEFAULT NULL,
  KEY (`ftime`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE (YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE = InnoDB,
 PARTITION p_2018 VALUES LESS THAN (2018) ENGINE = InnoDB,
 PARTITION p_2019 VALUES LESS THAN (2019) ENGINE = InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE = InnoDB);
insert into t values('2017-4-1',1),('2018-4-1',1);
```

在range分区中可以使用函数.**对于mysql，server端认为分区表是一个表，而存储引擎认为分区表示多个表**

对于分区表中数据的访问，要求是必须要有分区键的查询，如果没有的话mysql会一次性访问全部的分区。如果一个表分区过多的话(Too many open file)，所以一次性不能创建过多的分区

在顺丰中的时候会有定期创建分区和定期删除分区的操作

通过设置创建时有多少个分区，最多创建多少个分区来限定分区数量，根据设定的分区增加频率，定时去创建分区，找到当前分区的最大的值，然后在分区名称+1，然后判断分区数量，如果大于限定的分区数量的话就将最小的分区给drop 掉(分区命名规则p数字...)



分区使用场景：

分区表的好处是对于业务端来说是透明的，认为就是一个表.分区表一般运用在日志表，接口表等场景，可以按天来分或者是按月来分表
