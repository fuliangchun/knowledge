---
title: mysql索引
category: mysql
order: 3
---


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