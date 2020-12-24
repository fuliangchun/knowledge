---
title: mvcc
category: mysql
order: 2
---

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
![](https://fuliangchun.github.io/knowledge/images/mdl.png)

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

![](https://fuliangchun.github.io/knowledge/images/deadlock.png)

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

![](https://fuliangchun.github.io/knowledge/images/insertundolog.png)

* update Undo log  update类型的undo log和insert的不同，这种类型的Undolog需要提供MVCC的支持(可能多个事务在操作同一行数据)，因此不能再事务提交后就删除.需要后续等purge线程确定了没有多版本影响之后才进行删除。另外可以看出在记录undo log会记录旧的事务id，基于回滚指针，另外还会记录更新修改的列的Old Value，这些是用来进行回滚的.当有多个线程同时操作一条数据的时候，生成的undolog可以看成是一个链表.
![](https://fuliangchun.github.io/knowledge/images/updateundolog.png)

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



