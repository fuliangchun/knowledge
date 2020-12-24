---
title: iOS
category: mysql
order: 1
---

## mysql

### mysql架构

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-15 下午9.08.39.png" style="zoom:50%;" />

* server层
  * 连接器  处理连接，授权认证，安全等。wait_timeout 用来控制连接的空闲超时时间，默认值是 8 小时。现在应用都是长连接，如果发生mysql服务内存涨的特别快(连接占用的资源需要断开之后才会释放)
    * 定时断开长连接 或者长事务之后断开连接
    * 5.7以后可以每执行一个比较大的操作以后mysql_reset_connection，来将连接恢复到连接刚创建的状态
  * 缓存 缓存其实就是一个kv结构的数据，key为执行的sql语句，value为查询的结果。
    * 用于优化查询的效率，采用缓存(LRU)，如果发现有直接相同的sql直接返回，省略了解析器解析和优化的过程。**mysql中使用缓存弊大于利**，因为查询缓存失效过于频繁，只要对表进行更新，那么这个表上所有的查询缓存都需要清空。**mysql 8.0中已经将查询缓存去掉了**
  * 分析器  简单来说就是理解当前提交过来的sql语句是要干什么
    * 词法分析   将sql语句进行分析,将关键字识别，将字符串对应成表，查询字段
    * 语法分析   分析语法是否符合mysql规范，符合的话最后会解析成为一个语法树(you have an error in your SQL syntax)
  * 优化器   分析器是理解要做什么的话，优化器则是根据自己的分析来决定如何去做。负责生成执行计划，决定使用什么索引，决定表的连接顺序等。基于CBO(基于成本的一种优化)，例如执行计划中的rows字段就非常重要，mysql的选择执行计划会考虑到rows等一些因素
  * 执行器   获取到执行的计划，然后调用存储引擎的数据接口获取数据(此处其实会判断是否由查询表的权限),存储引擎是按页读取(16k),而执行器是一行一行的调用存储引擎的接口获取数据的

* 存储引擎  负责mysql数据的存储和提取。在mysql中存储引擎是插件式的。可以通过创建表的时候指定engine = innoDB来指定，默认其实就是InnoDB。tips：其实innoDB并不是mysql本身的，是其他公司开发的插件，最终被集成到了mysql中,这就是开源的好处啦

### SQL的执行顺序

#### 查询的执行流程

 查询的执行流程其实相对于简单。

* 由连接器处理连接(很多的client端(就是我们的应用)都采用连接池管理连接(长连接))，用户密码校验，权限校验，安全
* 8.0之前会去缓存中进行查询，若命中则直接返回，否则进行后续步骤
* 分析器 进行词法语法分析，理解sql语句的语义(语法树)
* 优化器根据选择一条自己认为最佳的执行计划
* 执行器按照执行计划去存储引擎中获取数据

#### 执行流程

事务的特点是ACID

* Atomic 原子性  事务中的所有的操作是原子的，不可分割，要么全部成功，要么都失败
* 一致性  从一个一致性状态变成另一个一致性状态
* 隔离性  事务之间是互不影响的
  * 读未提交 Read Uncommit
  * 读已提交 Read commit
  * 可重复读 Reaptable Read
  * 串行化 Serializable
* 持久性 一旦提交操作就永久保存在数据库中



**mysql中采用了一种WAL的思想(write-ahead logging),关键点就是先写日志，再写磁盘。** 为什么要这么做呢？

简单来说就是为了性能，如果每次更新操作都要写磁盘，然后磁盘也要找到那条记录，然后再更新(更新其实有可能设计到索引的修改,页分裂之类的，索引相对于写日志来说复杂程度高很多)。这么整个过程的IO成本(写日志是顺序写，而写到数据文件中就有可能不是顺序写了),查找成本都很高,为了解决这个问题,从而采用了WAL的思想来设计.

问题：mysql采用WAL，由原来的写一次磁盘变成写两次磁盘(redo log,binlog)？为什么说会更加快呢？

假设mysql执行dml语句的时候，会将操作的数据线读入内存中mysql的是先操作内存，在写日志的方式完成数据的更新的。内存中修改会非常快，然后再通过写日志的方式，保持这次更新不丢失.

* 对于redo log和binlog来说，写都是顺序写的,磁盘顺序写很快
* 同时在写入磁盘调用内核的fsync的时候会进行组提交策略，就是会将多个redo log的写合并到一起，一次性执行写入(所以实际的Io并不会比单次写数据库文件多很多)

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-19 下午5.08.08.png" style="zoom:33%;" />

##### Redo log

需要说明一下的就是redo log是inno DB存储引擎独有的**redo log是有固定大小的**,如下图，假设设置了配置为一组 4 个文件，每个文件的大小是 1GB的话.需要说一下的是redo log是循环写入的。每次触发一次从log文件flush到数据文件的操作就往后移动check point，而每次往日志文件中记录则会移动write pos，当二者接近的时候就会触发强制性的flush数据文件的操作。

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-15 下午9.38.56.png" style="zoom:40%;" />

有了redo log之后就可以保证数据库在发生异常重启的时候,之前提交的记录都不会丢失,crash-safe。也有了这种机制，所以Inno DB才实现了ACID中的持久性,以及协助实现了一致性

##### Binlog 

Redo log是引擎层的日志，而InnoDB可以说是server层的日志。所以的存储引擎在做更新操作的时候都会记录binlog中。说一下binlog是没有办法实现crash safe能力的.binlog是记录数据发生了什么变化，更新之前是什么样子更新之后是什么样子(项目中使用BInlog 解析来将更新的数据同步到es中)

**为什么说Binlog不能提供crash safe能力呢？**

假设没有redo log，那么保存数据的流程是 写Binlog->存数据  或者是存数据->写binlog，那么无论哪种在数据库崩溃的时候,那么无论如何利用binlog来恢复都会丢失或者多一些数据.不准确

binlog记录数据有三种模式：

ROW(以行数据的方式),STATEMENT(以执行sql语句的方式) MIXED(混合方式,普通情况下用statement，特殊情况使用row...)**一般推荐使用row**对于使用Binlog进行组成同步的场景下，如果使用statement的话有可能造成主从数据不一致的情况(例如会因为可能使用的索引列不同而执行了不同的执行计划导致操作的数据不同)。对于一些查询如果是用statemnt，Mysql会提示一个警告.这也是为什么会有Mixed这种隔离级别，其会在一些造成歧义的sql语句的时候采用row这种方式

binlog也时长用来做数据库的备份工作,主从之间的复制也是基于binlog来做的

在InnoDB中dml 的执行顺序

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-15 下午10.06.08.png" style="zoom:50%;" />

上面浅色的内容是在引擎层执行的，深色的是在server层执行的。**注意更新的时候是首先将数据更新到引擎的内存中的**

* 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
* 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
* 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
* 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
* 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

##### 脏页

如上面的流程图，当我们在执行upate t set name =xx where id =2，首先判断内存中是否有这行数据，没有的话会去读取数据到内存中，这个读取过程是按页读取(16k)，读取的是一行行的数据,后续更新数据也是更新内存，然后记录日志就算更新完了.**后续如果涉及到这个表的查询其实也查询的是查询的内存**,当数据在内存中不存在才会通过树进行查询.

如果对于内存中的数据进行了更新，这个页就是脏页,**即使是涉及到修改了数据,会变动索引之类的数据也不会里面就更新到数据里面**

问题1：什么时候才会将脏页数据刷到数据中?

* 当 redo log写满了。这时候系统就会停止所有的更新操作，将更新的这部分日志对应的脏页同步到磁盘中，此时所有的更新全部停止，此时写的性能变为0，必须待刷一部分脏页后才能更新,这时就会导致 sql语句 执行的很慢。

* 也可能是系统内存不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，则需要先将脏页同步到磁盘,空出来的给别的数据页使用。

* MySQL 认为系统“空闲”的时候,反正闲着也是闲着反正有机会就同步到磁盘一些数据

* MySQL 正常关闭。这时候，MySQL 会把内存的脏页都同步到磁盘上，这样下次 MySQL 启动的时候，就可以直接从磁盘上读数据，启动速度会很快。

问题2：脏页写入数据库文件中，会引起页分裂或者页合并

如果脏页中有涉及到索引的变动，则就有可能引起页的合并等操作(例如删除了很多的数据...)

**redo log的两阶段提交**

为什么需要两阶段提交？

因为mysql需要redo log和Binlog处于一致状态，首先写入redo log并处于prepare状态,再写入BInlog，最后将redo log中变成commit状态，最终完成事务的提交.

两阶段也被用来做分布式事务的一个处理方式.即只有在多个都进入了prepare阶段之后，最终所有的事务一起完成提交.

InnoDB从异常中恢复其实默认就会读取redo log和binlog，然后判断其中的状态

* 如果数据在redo log中处于commit状态，则无须从BInlog中去寻找了
* redo log处于prepare状态，binlog中无数据，说明事务未提交就崩溃了 则回滚这个事务
* redo log 处于prepare状态，但是binlog中有数据，则提交这个事务



<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午3.33.45.png" style="zoom:33%;" />

事务的执行流程：

* 如果数据页不在InnoDB的 buffer pool中的话需要数据读入buffer pool
* 更新数据页数据(脏页),原数据写入undo log(原子性)用于回滚，在合适的时候由后台purge线程删除
* 将更新的操作记录到写入redo log buffer中,默认是innodb_flush_log_trx_commit(默认是1，即事务提交就刷盘),但是由于组提交(即会稍微延后调动内核fsync,多个日志一次性提交),此时**redo log的状态为prepare**
* 此时server端收到执行完成消息，然后进行binlog写入，binlog也是先写到Binlog buffer中，不同于redo log buffer是所有线程公用的，binlog buffer是线程独有的(为了保证Binlog的完整性),根据sync_binlog的设置(默认是1，事务提交就里面写入磁盘),这里也会有组提交优化.
* 当Binlog写入完成则将redo log变成commit状态，最终提交事务完成

因为redo log是先写到redo log buffer中，而这里面又有组提交等概念和后台线程定时刷等，则有可能事务还未提交，其部分redo日志事务还未提交，其实日志就有可能提交到磁盘文件中了。如果后续事务回滚，其实也没有影响

### 锁与MVCC

#### 锁

mysql中的锁分为三种，全局锁，表锁，行锁

##### 全局锁

对整个数据库实例进行加锁，Flush tables with read lock (FTWRL)。会对整个数据库的实例处于只读状态，会阻塞**dml,ddl,以及修改事务的提交**,通常运用在全库进行备份的时候.

##### 表锁

MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。

表锁的语法是lock tables ...read/write..线程1执行 lock tables t1 read, t2 write; 则其他线程对于t1的写，和t2的读写都讲会阻塞..同时线程1也只能对t1进行读，t2进行读写.在没有更细界别的锁之前，表锁是一种常用处理并发的模式

**MDL表锁**

MDL加锁是一种默认的行为，主要是为了解决在进行select或者dml操作的时候有其他的线程并发进行DDL的操作

即select和dml操作会给表加上dml读锁，而ddl操作会加上DML写锁。

DML读写和DML写写是互斥的。这很容易理解。同时也需要注意的一点就是不要在业务高峰的时候执行DDL操作，因为这会给表加上MDL表锁，这意味着在DDL操作之后的请求都会被阻塞.

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-19 下午3.43.27.png" style="zoom:50%;" />

**如何安全的给表加字段**

* 判断当前是否由长事务在执行，如果有的话可以等长事务执行完或者kill掉长事务再执行

* 如果是个热点表加字段可以指定多长时间如果能拿到mdl锁的话就进行alter 操作，如果拿不到的话就不进行操作

  ```sql
  ALTER TABLE tbl_name NOWAIT add column ...
  ALTER TABLE tbl_name WAIT N add column ... 
  ```

  

##### 行锁

InnoDB支持行级锁。

* 共享锁 读锁是共享的，读锁与读锁之间是不互斥的不阻塞的。
* 排他锁  写锁是排他的，一个写锁会阻塞其他的读锁和排他锁

需要注意的是**InnoDB事务中，是在需要的时候才加上的，但是是要在事务提交的时候才会释放的**,如果事务中需要对多个行进行加锁，那么就可以将最有可能造成锁冲突，最有可能影响并发的所放在最后面执行.例如在购买场景中，可以将商户的账户的扣款放在用户的扣款之后，因为商户的账户可能会有多个事务并发访问.如果在长事务中，占有锁过多的话，从而影响了数据库的并发性能

**死锁**

当系统中多个并发线程出现了循环依赖的话就会引发死锁的问题

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-19 下午3.55.27.png" style="zoom:50%;" />

在事务A中拿到了id=1的行锁，需要获得id=2的行锁，事务B中拿到了id=2的行锁，需要获得id=1的行锁，从而造成了死锁

处理死锁的两种策略

* 直接进入等待，设置超时时间，这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置。默认50s
* 发起死锁检测，主动回滚死锁链条中的某一个事务,让其他事务得以执行，默认打开

死锁检测是一种很好的机制，但是其也是有弊端的,会消耗cpu的性能，最明显的是对一张热点表，多个事务更新同一行数据，每个事务都检测锁有没有被其他线程加锁，这个判断过程很耗费cpu.但是对于并发度不高的应用，死锁检测还是一种比较好的策略.

##### 行锁的具体实现

- Record Lock:单个记录上的锁 索引记录。如果没有索引的话会采用隐式的主键来进行锁定。·
- Gap Lock:间隙锁，锁定一个范围，但不包含本身
- Next-key Lock:Record Lock+Gap Lock,锁定一个范围，并且锁定记录本身。
  Next-Key Lock的表现形式是前开后闭的形式。
  例如索引有10,11,13,20的话那么可能被Next-KEY Lock的区间为：（负无穷,10],(10,11],(11,13],(13,20],(20,正无穷）..
  然而当我们查询的索引含有唯一属性的时,InnoDb存储引擎会对Next-Key lock进行优化，将其降低为Record Lock，即仅锁住索引本身，而不是范围。

间隙锁的出现是为了解决幻读的问题

**需要注意的是mysql其实是在索引上加锁的**下面用示例来说明

```sql
CREATE TABLE z (
  id INT PRIMARY KEY AUTO_INCREMENT,
  b  INT,
  KEY b(b)
)
  ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

INSERT INTO z (id, b)
VALUES (1, 2),
  (3, 4),
  (5, 6),
  (7, 8),
  (9, 10);
  
-- SESSION A:
BEGIN;
SELECT *
FROM z
WHERE b = 6 FOR UPDATE;

-- session B
INSERT INTO z VALUES (2, 4);/*success*/
INSERT INTO z VALUES (2, 8);/*blocked*/
INSERT INTO z VALUES (4, 4);/*blocked*/
INSERT INTO z VALUES (4, 8);/*blocked*/
INSERT INTO z VALUES (8, 4);/*blocked*/
INSERT INTO z VALUES (8, 8);/*success*/
INSERT INTO z VALUES (0, 4);/*blocked*/
INSERT INTO z VALUES (-1, 4);/*success*/
```

可以看下为什么session B执行的结果是这样的?

sessionA对b=6进行加锁，b是索引列，其实是对b的这个b+树进行加锁，最终会对(4,8)进行加锁.准确来说是会读

{(3,4),(7,8)}这一个区间进行加锁，即后续的操作都不允许在这中间进行插入和修改。在非聚簇索引中，索引的结构是索引列相同，主键是递增的

* (2,4)如果插入数据的话会在区间因为则可以插入
* (2,8)阻塞，因为其会插入到(7,8)之前，所以不允许插入
* (4,4)阻塞，因为会插入到(3,4)以后，所以不允许插入
* (4,8)阻塞，因为会插入(7,8)之前，不允许插入
* (8,4)阻塞，会插入到(3,4)以后，不允许插入
* (8,8)允许插入,因为会插入到(7,8)的后面
* (0,4)阻塞，**在Mysql中如主键写0或者Null，会设置为递增的主键值即8,4**
* （-1，4）允许插入，会插入到(3,4)以前





#### MVCC

在Mysql中虽然有共享锁和排他锁来保证并发时的数据正确，但这样加锁的访问真正意义上并不是并发，或者只是并发读，因为它最终实现的是读写串行化，这样大大的降低了数据库的读写性能。为了进一步提高并发的性能，mysql基于MVCC(Multi-Version Concurrency Control，即多版本并发控制)做了进一步的优化，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读。

##### mysql中的读

分为两种:

* 当前读  像select lock in share mode(共享锁), select for update ; update, insert ,delete(排他锁)这些操作都是一种当前读，为什么叫当前读？就是它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。
* 快照读 即不加锁非阻塞的读，在非串行化的隔离级别中存在。快照读读取的是记录的可见的版本(不一定是最新的数据)，这种做的好处是读避免了加锁的操作，节约了性能

##### 当前读，快照读和MVCC的关系

 MVCC多版本并发控制的思想其实就是要实现一个数据同一个时刻多个版本，使得读写没有冲突。而快照读就是实现的一个非阻塞读功能，而当前读就是悲观锁的一种实现(读写都进行加锁)

##### 实现原理

mysql实现MVCC的依赖Undo log、readView,隐藏列

###### 隐藏列

`InnoDB`向数据库中存储的每一行添加三个字段。`DB_TRX_ID`字段表示插入或更新该行的最后一个事务的事务标识符。此外，删除在内部被视为更新，在该更新中，行中的特殊位被设置为将其标记为已删除。

 `DB_ROLL_PTR`字段，称为滚动指针。回滚指针指向写入回滚段的undo log。如果行已更新，则undo log记录将包含在更新行之前重建行内容所必需的信息。

`DB_ROW_ID`字段包含一个行ID，该行ID随着插入新行而单调增加。如果 `InnoDB`自动生成聚集索引，该索引包含行ID值。否则，该 `DB_ROW_ID`列不会出现在任何索引中。

###### Undo log

Undo log有两种 

* insert Undo log    只对本事务可见，对其他事务不可见(事务隔离性要求),这种undo log在事务提交后直接删除，不需要purge线程进行删除。其中没有记录只是记录了表名 与一些唯一键字段

  <img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午3.11.31.png" style="zoom:50%;" />

* update Undo log  update类型的undo log和insert的不同，这种类型的Undolog需要提供MVCC的支持(可能多个事务在操作同一行数据)，因此不能再事务提交后就删除.需要后续等purge线程确定了没有多版本影响之后才进行删除。另外可以看出在记录undo log会记录旧的事务id，基于回滚指针，另外还会记录更新修改的列的Old Value，这些是用来进行回滚的.当有多个线程同时操作一条数据的时候，生成的undolog可以看成是一个链表.

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午3.13.04.png" style="zoom:50%;" />

Undo log通常是用来为我们做事务的一致性的。在修改数据的时候记录undo log，当事务进行回滚，找到对应的undo log记录，可以获得数据的历史的值.可以理解 事务回滚就是一条反向的执行语句 insert->delete，delete-> insert 。。。。。



###### RedView

Read View就是事务进行快照读操作的时候生产的读视图(Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的ID集合(当每个事务开启时，都会被分配一个ID, 这个ID是递增的，所以最新的事务，ID值越大),同时记录最大值和最小值

假设 id 集合为 ids   最小值为min  最大值为max

先查询当前记录的线程为X

* 若X<min 则表示,则表示数据是在生成readView直接就提交了的，对当前事务是可见的
* 若 X大于max 则表示数据是在生成之后才提交的，所以数据对于当前事务不可见的
* 再判断是否在ids中，如果在的话则表示生成readView的时候还未提交，则最新数据对于当前事务不可见，如果不在的话，则表示生成readView时候已经提交了，对于当前事务可见

如果对于当前事务不可见的话，就需要依赖undo log了，找到一个可见的版本用于显示。



已提交读(Read Commited)和可重复度(Repeatable Read)两个事务隔离级别的区别在于生成readView的时机.读已提交时在每次读取数据的时候都生成一个ReadView，所以两次读的时候可能会产生不一样的结果 出现了不可重复读的问题。而可重复读的话多次读只会生成一个ReadView，所以不会有不可重复读的问题.

其实按照上面这个理论,利用MVCC其实是可以解决幻读的，或者是解决了快照读的幻读。但是对于当前读，其不依赖于MVCC，所以不能解决幻读,当前读的话是需要幻读解决的.

脏读： 能够读取到其他事务未提交到的数据  隔离级别(Read Uncommit)会产生

不可重复读：对于同一条数据，在同一个事务中读取多次的值不一样(中间有可能是其他事务修改提交了)隔离级别(Read Commited)会产生

幻读：对于执行相同条件的sql，执行多次得到的数据条数不一样(其他事务插入了) .

利用MVCC可以解决掉快照读的幻读的问题，但是解决不了当前读的幻读问题。其实依靠gap lock来实现的.







### Mysql 索引

Mysql一般常用的索引类型是B+ tree和hash和fulltext(全文检索)。在InnoDB数据库中支持

* B+ tree索引
* Full-text 索引
* Hash索引(不支持手动创建,但是内部将哈希索引用于其自适应哈希索引功能,InnoDB发现某些索引被引用的非常频繁的时候会在b+树索引之上在索引关键字的前缀构建哈希索引,方便快速定位,但是这个完全是一个内部自动的行为,并不是我们在索引的时候选择hash)

#### Hash索引

hash索引顾名思义是用hash值来实现快速定位的一种优化的数据结构，其对于等值查询非常快速，但是对于范围查找等功能不适用,另外如果hash冲突发生过多的话，对于索引的维护也会比较消耗性能，不能做排序操作,hash索引中只包含hash值与指针，如果是在内存中的话是很快的(memory是hash索引),但是Innodb等一些持久化的数据的话还是限制条件过多，不适合

#### B+ 树索引

InnoDB中的索引一般都指的是B+ 树索引.

**为什么要使用B+树来做索引呢?**
索引主要的作用是为了加速查询效率.并且需要兼顾在数据存储的时候不需要耗费过多的性能.像普通的二叉树或者AVL树或者红黑树其实都可以用来提高查询的效率

* 普通二叉树 会有极端情况下形成一条单链，且二叉树数据多的时候深度过高
* AVL树 是一种优化过后的二叉树，其可以在数据插入过程中维持平衡(左子树和右子树相差不能超过1)缺点是数据插入过程中移动频繁，数据多深度过高
* 红黑树 不像AVL那样追求绝对平衡，最求一种相对平衡，两个子树之间的高度不能超过一半。通过结点颜色变化来做的(两个红色的结点不能相连，每个子树的黑色结点个数相同) 确定是数据多 树的深度过高
  <img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午11.11.06.png" style="zoom:50%;" />

这几种数据结构都有一个严重的问题就是树的深度过深而造成树的深度过高，从而影响查询效率.所以mysql采用B树(B树的变种)的结构来索引。

**B树索引的特点**

* 所有的键值都保存在树中(准确来说是每个结点中都保存着m个数据)

* 每个结点都有M个子节点  M是由单个B树的阶(degree)来决定的，一般的可以认定为一个内存页(4k/16k)能够保存的数据个数来决定的
* 叶子结点都在同一层

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午11.32.32.png" style="zoom:50%;" />



InnoDB对B数做了一层优化

如果有B树来做索引的话有几个问题，一个是因为结点中保存数据，会限制B树的阶，导致树长度增加.二是因为范围查询的时候需要返回父结点，增加io次数



**B+树的特点**

* 非叶子节点只存储key，叶子结点存储key+数据。这么做的好处是提升了B+树的阶(一个节点可以存储更多的key与指针),即一个结点可以拥有更多的叶子结点，从而降低B+树的高度,提升查询性能。100万级的数据可能最终的树高度就3-4层，3次io就可以查询到数据
* 叶子结点的指针相互连接，类似于一个双向链表，范围查找或者顺序读的时候会非常快

![](/Users/fuliangchun/Desktop/屏幕快照 2020-12-12 下午11.49.14.png)

**InnoDB和MySIAM B+树区别**

* MySIAM中的B+树 叶子节点中存储的不是数据，而是一个指向数据的指针(从文件中就可以看出,MySIAM一张表一般会生成3个文件,一个表的元数据,一个表的索引，一个表的数据)。对于聚簇索引和非聚簇索引来说存储是一样的。
* InnoDB中B+树 叶子节点中保存的是完整的数据(聚簇索引),而非聚簇索引叶子结点保存的是聚簇索引的索引键(一般情况下是主键id)



#### 执行计划

索引是辅助提高查询效率的，我们一般情况下在创建表的时候会创建一部分索引，但是会有某些情况下查询并没有走索引(或者是sql执行效率比较低(所谓的慢sql,一般由运维或者DBA找出执行慢的sql交由开发去修改))。我们往往会通过查看执行计划来分析.

可以使用explain+SQL语句来模拟优化器执行SQL查询语句，从而知道mysql是如何处理sql语句的。

##### 执行计划中包含的信息

|    Column     |                    Meaning                     |
| :-----------: | :--------------------------------------------: |
|      id       |            The `SELECT` identifier             |
|  select_type  |               The `SELECT` type                |
|     table     |          The table for the output row          |
|  partitions   |            The matching partitions             |
|     type      |                 The join type                  |
| possible_keys |         The possible indexes to choose         |
|      key      |           The index actually chosen            |
|    key_len    |          The length of the chosen key          |
|      ref      |       The columns compared to the index        |
|     rows      |        Estimate of rows to be examined         |
|   filtered    | Percentage of rows filtered by table condition |
|     extra     |             Additional information             |

* id  表示select查询的序列号，包含一组数字，表示查询中执行select子句或者操作表的顺序。id越大优先级越高，越先执行，id相同则由上往下执行

* Select_type 主要用来分辨查询的类型，是普通查询还是联合查询还是子查询

  |         类型         |                             含义                             |
  | :------------------: | :----------------------------------------------------------: |
  |        SIMPLE        |              简单查询，没有使用union或者子查询               |
  |       PRIMARY        |                  子查询外层的查询为PRIMARY                   |
  |        UNION         |                       使用了union查询                        |
  |   DEPENDENT UNION    | Second or later SELECT statement in a UNION, dependent on outer query |
  |     UNION RESULT     |                       union获取的结果                        |
  |       SUBQUERY       |                   First SELECT in subquery                   |
  |  DEPENDENT SUBQUERY  |      First SELECT in subquery, dependent on outer query      |
  |       DERIVED        |            rom子句中出现的子查询，也叫做派生类，             |
  | UNCACHEABLE SUBQUERY | A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query |
  |  UNCACHEABLE UNION   | The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY) |

* table 对应行正在访问哪一个表，表名或者别名，可能是临时表或者union合并结果集

* Type  type显示的是访问类型，访问类型表示我是以何种方式去访问我们的数据，访问的类型有很多，效率从最好到最坏依次是：

  system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL 

  一般情况下，得保证查询至少达到range级别，最好能达到ref

  |      type       |                             含义                             |
  | :-------------: | :----------------------------------------------------------: |
  |       ALL       |                  全表扫描（没有使用到索引）                  |
  |      index      |                   遍历索引（只查询索引列）                   |
  |      range      |              索引范围查找(在索引列添加范围查询)              |
  | index_subquery  |                      在子查询中使用 ref                      |
  | unique_subquery |                    在子查询中使用 eq_ref                     |
  |   ref_or_null   |                  对Null进行索引的优化的 ref                  |
  |    fulltext     |                         使用全文索引                         |
  |       ref       |                    使用非唯一索引查找数据                    |
  |     eq_ref      | 在join查询中使用 PRIMARY KEY or UNIQUE NOT NULL索引关联 (这里需要把address的user_id设置为unique类型的索引) |
  |      const      |        使用主键或者唯一索引，且匹配的结果只有一条记录        |
  |  system const   |        连接类型的特例，查询的表为系统表或通过主键查询        |

* possible_keys     sql可能用到的索引

* key  	实际使用的索引，如果为null，则没有使用索引

* key_len   表示索引中使用的字节数，可以通过key_len计算查询中使用的索引长度，在不损失精度的情况下长度越短越好。

* ref  显示索引的哪一列被使用了，如果可能的话，是一个常数

* rows   根据表的统计信息及索引使用情况，大致估算出找出所需记录需要读取的行数，此参数很重要，直接反应的sql找了多少数据，在完成目的的情况下越少越好。如果预估的行数太多，可能会放弃使用索引而采用全表扫描的方式

* extra 额外信息

```sql
--using filesort:说明mysql无法利用索引进行排序，只能利用排序算法进行排序，会消耗额外的位置
explain select * from emp order by sal;

--using temporary:建立临时表来保存中间结果，查询完成之后把临时表删除
explain select ename,count(*) from emp where deptno = 10 group by ename;

--using index:这个表示当前的查询时覆盖索引的，直接从索引中读取数据，而不用访问数据表。如果同时出现using where 表名索引被用来执行索引键值的查找，如果没有，表面索引被用来读取数据，而不是真的查找
explain select deptno,count(*) from emp group by deptno limit 10;

--using where:使用where进行条件过滤
explain select * from t_user where id = 1;

--using join buffer:使用连接缓存，情况没有模拟出来

--impossible where：where语句的结果总是false
explain select * from emp where empno = 7469;

--using index condition
--没有索引下推场景，存储引擎将遍历索引以找到对应的数据，并将其返回给MySQL服务器，mysql服务器再通过WHERE条件过滤数据。使用索引下推的话就则在找到数据的时候就将where条件给过滤
explain SELECT * FROM people  WHERE zipcode='95054' AND lastname LIKE '%etrunia%';

```

#### 索引优化

##### 索引的基础知识

**索引分类**

* 主键索引 (又叫做聚簇索引,数据是保存在这个数的叶子节点上的) ，主键索引一般都会设置为自增的，这种的话是顺序追加的，可以避免页分裂的问题
* 唯一索引 (也是非聚簇索引,但是其中不允许重复)
* 普通索引(非聚簇索引,索引值允许重复)
* 组合索引(非聚簇索引,不同的是索引的值允许有多个,遵循最左匹配原则)

**名词**

回表：在使用非聚簇索引(非聚簇索引保存的是索引的键值以及聚簇索引的键值(一般是主键)),如果需要数据(非覆盖索引的情况)，需要通过聚簇索引的键值去聚簇索引中进行再次查询(如果是能够使用覆盖索引的话就省略了一次回表的过程)

覆盖索引：即查询非聚簇索引的时候，select中的字段是索引的键值(或者是键值+主键Id)，这样可以省略一次回表,在执行计划中的体现是在extra信息中显示using index

最左匹配:在使用组合索引的时候,非聚簇索引键值是多个字段.所以在查询的时候，假设有3个字段的组合索引(a,b,c)，那么使用a=1 and c=1的时候 其实是只用到了索引中的a的部分

索引下推：当我们通过索引进行检索的时候，如果还有其他非索引字段的where 条件，会在引擎层就进行过滤，以前是查询出来，通过mysql server进行过滤的

##### 组合索引

组合索引中有些场景其实是会使索引失效的

| 语句                               | 索引是否失效                                                 |
| ---------------------------------- | ------------------------------------------------------------ |
| where a=3                          | 只使用了a(可以通过explain中key_len知道)                      |
| Where a=3 and b=4                  | 只使用了a,b                                                  |
| where a=3 and b=4 and c=5          | 使用了abc                                                    |
| where a= 3 and c=5                 | 只使用了a                                                    |
| where b=4 or c=5                   | 没有使用索引                                                 |
| where a=3 and b>4 and c=5          | 只使用过了ab但是会使用索引下推(extra  using index condition) |
| Where a>3 and b=4                  | 只能使用a 同样会有索引下推                                   |
| where a like 'aa%' and b=4 and c=5 | 只使用a                                                      |

可以看出组合索引 的场景组合索引定义的顺序是有关系的.如果不考虑实际场景的查询，组合索引的顺序可以根据键值的长度定，小的在前面，这样做的目的是一个页可以存储更多的数据

##### 组合索引如何选择字段顺序

* 考虑索引的复用能力。因为可以支持最左前缀，那么例如(a,b)组合索引的话，其实就可以省略一个a的索引了即**如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的**
* 考虑空间的问题,因为一个索引字段越大的话，其有可能会造成树的高度就越高，导致增加Io量。如果创建了(a,b)索引，如果有遇到通过b查询的 也可以再创建一个b的索引

##### 优化tips

* 索引列查询尽量不要使用表达式  把计算放在业务层而不是数据层  where id =5  vs where id =4+1

* 尽量使用主键索引,减少回表

* 使用前缀索引   如果需要索引很长的字符串,这会降低查询的性能(因为树可能会变高,io增多，可以使用列的部分来做索引,但是需要根据实际业务来决定索引的长度  可以根据真实数据的重复读来设置合适的

  ```sql
  -- 判断合适的长度
  select count(distinct left(city,3))/count(*) as sel3,
  count(distinct left(city,4))/count(*) as sel4,
  count(distinct left(city,5))/count(*) as sel5,
  count(distinct left(city,6))/count(*) as sel6,
  count(distinct left(city,7))/count(*) as sel7,
  count(distinct left(city,8))/count(*) as sel8 
  from citydemo;
  --设置索引
  alter table citydemo add key(city(7)); 
  ```

* 可以使用索引来帮助排序

  mysql中两种排序方式：

  * 使用索引来辅助排序 这种方式利用了索引自身的有序来帮助排序

    *  where a =3 order by b,c (这种会在b,c之前填充一个常量，所以也是符合的)
    * where a =3 order by a,b,c

  * 使用文件排序算法排序。体现是explain 的时候 extra信息为 using filesort。

    * where a =3 order by c  

    * where a=3 order by b,c,d  

    * where a =3 order by b desc c asc 

    * where a =3 order by d 

  什么时候才会使用索引排序,order by子句的顺序完全一致，并且所有列的排序方式都一样时()

* union和union all都会使用索引,但是推荐使用in

* 隐式转换会导致索引失效   where name ='123' (失效) 和 name = 123(失效)

* 更新特别频繁，区分度不是很高的字段上不宜创建索引

* 推荐索引列不能为空(null其实也能被索引,但是null =null 为false 要小心)

* 单表索引控制在5个以内

* 单个索引的字段不要超过5个

#### mysql优化器选择错误的执行计划原因

优化器是找到一个最优的执行方案，并以最小的代价去执行(CBO).其中一个重要的因素就是扫描行数(explain 中的rows)，行数扫描的少则意味着更少的IO.除了扫描行数其他的是否使用临时表是否进行排序等操作.

**扫描行数如何判断?**

mysql中的扫描行数是根据统计信息进行估算的.统计信息其实就是索引的区分度,一个索引上不同的值越多，这个索引的区分度就越好,这个区分度也叫做基数”（cardinality）。show index from table 可以看到一个表中索引的区分度。这个值其实是通过采样值来计算出来的，获取n个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。

![](/Users/fuliangchun/Downloads/16dbf8124ad529fec0066950446079d4.png)

统计的基数是有可能不准，还有一些场景即使是基数小，mysql也是有可能去选择大的基数对应的索引的。例如如果非聚簇索引需要回表查询这个也是需要计算在内的，优化器回顾及这些代价，如果刚好某个索引中对应到了order by 中的字段,那么也会计算在内

**如何处理异常索引的情况**

* 在某些情况下可以使用select * from t force index(idx_xxx)  where a=xx and ....

* 可以新建一个更加合适的索引，给优化器进行选择或者是删除掉误用的索引(有可能该索引就不应该存在)
* 可以修改语句引导mysql选择我们期望的索引

```sql
mysql> explain select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1;

-- 理论上使用a的索引会更加合适，但是mysql判断where中有b且用了b做排序，则会选择b，即使b对应的基数很大.
-- 可以通过改写order by b,a这种方式，就会引导mysql选择a索引.
mysql> explain select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b,a limit 1;
```



#### 查询优化

##### 优化数据访问

**减少查询不必要的列**

在生产中不用采用select * 的方式，查询需要多少数据就查询多少数据.这样可以替身查询性能

**采用预编译**

sql执行会经过语义分析语义优化制定执行计划等过程，如果一个sql频繁被执行，那么其实也是非常耗时的，可能sql只是参数不同,所以通过预编译的方式将这类语句的通过占位符的方式进行预编译，就省略了语义分析和语义优化的过程,从而提升性能。另外还可以防止sql注入

经典的一个场景就是批量插入,insert  values(),(),(), 通过foreach values的方式，这种方式其实很快，但是这种语句不会被mysql预编译缓存，所以可以使用Executor.Batch方式进一步提升插入性能.

#### mysql优化器选择错误的执行计划原因

mysql是根据其内部的统计信息来选择最优的执行计划的

* 统计数据不准确
* 执行计划成本不等于实际的成本
* mysql考虑的最优与我们的最优不同
* mysql不考虑其他并发的情况
* mysql不会考虑不受其控制的成本
  * 执行计划
  * 自定义函数

##### 关联查询

mysql中有三种join方式

* Simple Nested-Loop Join  这种是没有非索引字段的join  
  * 原理是取出驱动表的每行记录都去与被驱动表进行关联匹配,性能很差

* Index Nested-Loop Join  索引字段的Join
  * 通过索引来减少比较,驱动表通过索引找到对应的数据，然后进行回表，非驱动表如果关联是主键则非常快，非主键的话需要进行回表(进行非聚簇索引扫描,然后进行回表)
* Block Nested-Loop Join
  * 比Index Nested-Loop Join 多了一层join buffer，其会将驱动表的所有和join相关的列(所有参与查询的列)都放到Join buffer中，然后通过Join buffer在和 被驱动表进行关联.从而提升性能(一般如果join的字段没有索引的话会采用这种方式)

##### 特定类型的查询优化

**优化关联**

* 在适用关联的时候需要确保关联的字段上要有索引(一般来说只有被驱动表的关联的列要有索引，驱动表如果不会被其他表Join的话可以不创建索引)
* Group by ,ordr by 中确保只有一个表中的列,这样才有可能使用索引
* 子查询如果能用关联查询的话就尽量用关联查询(子查询会创建临时表，而关联查询不会)
* 

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



<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-19 下午5.34.08.png" style="zoom:50%;" />

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

<img src="/Users/fuliangchun/Desktop/屏幕快照 2020-12-19 下午5.59.30.png" style="zoom:50%;" />





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
