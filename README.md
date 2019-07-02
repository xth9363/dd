# Mysql事务并发问题和隔离级别

## 数据并发操作带来的问题

一个数据库可能拥有多个访问客户端，这些客户端都可以并发方式访问数据库。数据库中的相同数据可能同时被多个事务访问，如果没有采取必要的隔离措施，就会导致各种并发问题，破坏数据的完整性。

**1.丢失修改(Lost Update)**
当两个事务同时尝试去更新某一条数据记录时，就肯定会存在一个先一个后。而当事务A更新时，事务A还没提交，事务B就也过来进行更新，覆盖了事务A提交的更新数据.
```
第一类丢失更新
A事务撤销时，把已经提交的B事务的更新数据覆盖了。
```
![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/rshrta.jpg)
```
A事务在撤销时，“不小心”将B事务已经转入账户的金额给抹去了。
```
```
第二类丢失更新 
A事务覆盖B事务已经提交的数据，造成B事务所做操作丢失：
```
![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/kswfgi.jpg)


**2、脏读（dirty read）:**
```
A事务读取B事务尚未提交的更改数据，并在这个数据的基础上操作。
如果恰巧B事务回滚，那么A事务读到的数据根本是不被承认的。
来看取款事务和转账事务并发时引发的脏读场景：  
```
![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/wdximn.jpg)


**3、不可重复读（non-repeatable read）**
```
不可重复读是指一个事务范围内，多次查询某个数据，却得到不同的结果。

在第一个事务中的两次读取数据之间，由于第二个事务的修改，第一个事务两次读到的数据可能就是不一样的。

假设A在取款事务的过程中，B往该账户转账100元，A两次读取账户的余额发生不一致：
```
![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/gvpeok.jpg)


**4、幻读（phantom read）**
```
一个事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据，这种现象就称为“幻读”。

幻读一般发生在计算统计数据的事务中.
举一个例子，假设银行系统在同一个事务中，两次统计存款账户的总金额，
在两次统计过程中，刚好新增了一个存款账户，并存入100元，
这时，两次统计的总金额将不一致：
```
![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/mwlwgp.jpg)

### 事务的隔离级别

**Read uncommitted (读未提交)**:
最低的事务隔离级别，一个事务还没提交时，它做的变更就能被别的事务看到。

解决问题：丢失修改

**Read committed (读已提交)**:
保证一个事物提交后才能被另外一个事务读取。另外一个事务不能读取该事物未提交的数据。

解决问题：丢失修改，脏读

**Repeatable read (可重复读)**：
多次读取同一范围的数据会返回第一次查询的快照，即使其他事务对该数据做了更新修改。事务在执行期间看到的数据前后必须是一致的。

但如果这个事务在读取某个范围内的记录时，其他事务又在该范围内插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行，这就是幻读。

解决问题：丢失修改，脏读，不可重复读

**Serializable（串行化）**:

最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。

“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

事务 100% 隔离，可避免脏读、不可重复读、幻读的发生。

这个级别类似于可重复读，但是比可重复读更严格，InnoDB隐式地将所有普通SELECT语句转换为 SELECT ... LOCK IN SHARE MODE，即对普通的读取操作也进行加锁


![avatar](http://www.xiatianhao.com/media/article_images/2019/06/23/adbpmr.jpg)

看一个例子

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/23/1.jpg)

我们再举个例子

A，B 两个事务，分别做了一些操作，操作过程中，在不同隔离级别下查看变量的值：

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/23/2.jpg)


```
事物隔离级别越高，在并发下会产生的问题就越少，但同时付出的性能消耗也将越大，因此很多时候必须在并发性和性能之间做一个权衡。
所以设立了几种事物隔离级别，以便让不同的项目可以根据自己项目的并发情况选择合适的事物隔离级别，对于在事物隔离级别之外会产生的并发问题，在代码中做补偿。
```

#### 读未提交 级别 如何解决 丢失修改

```
如果多个线程同时修改共享数据，那么数据将会产生错乱，
可以使用加锁的方式对访问和操作共享数据的代码段（称做临界区）进行加锁，
使得同一时间只能有一个线程持有锁，达到保护共享数据的目的。
容易想到的是，InnoDB会对多个事务同时进行更改的数据进行加锁，以避免并发问题保证同步.
```
```
-- 事务1
begin;
Query OK, 0 rows affected (0.00 sec)

select * from t
;
+----+------+
| id | v    |
+----+------+
|  9 | a    |
+----+------+
1 row in set (0.01 sec)

update t set v=
1 where id=9;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
```
-- 事务2
begin;
Query OK, 0 rows affected (0.00 sec)

select * from t
;
+----+------+
| id | v    |
+----+------+
|  9 | a    |
+----+------+
1 row in set (0.00 sec)

update t set v=
2 where id=9;
-- 阻塞
```
```
-- 查看
select * from information_schema.innodb_locks; 
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| lock_id       | lock_trx_id | lock_mode | lock_type | lock_table | lock_index | lock_space | lock_page | lock_rec | lock_data |
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| 17498:285:3:2 | 17498       | X         | RECORD    | `x`.`t`    | PRIMARY    |        285 |         3 |        2 | 9         |
| 17497:285:3:2 | 17497       | X         | RECORD    | `x`.`t`    | PRIMARY    |        285 |         3 |        2 | 9         |
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
2 rows in set, 1 warning (0.00 sec)


四个隔离级别都会在UPDATE,INSERT,DELETE时给数据上写锁,所以都不会出现丢失修改的情况

```

### 读已提交 级别 如何解决 脏读

如何用锁解决脏读:在查询时加S锁,如果该行数据已有其他未提交事务修改,必然已经被加上X锁,阻塞等待释放.

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/23/2.jpg)

#### MVCC：Multi-Version Concurrent Control 多版本并发控制
MVCC是为了实现数据库的并发控制而设计的一种协议。从我们的直观理解上来看，要实现数据库的并发访问控制，最简单的做法就是加锁访问，即读的时候不能写（允许多个西线程同时读，即共享锁，S锁），写的时候不能读（一次最多只能有一个线程对同一份数据进行写操作，即排它锁，X锁）。这样的加锁访问，其实并不算是真正的并发，或者说它只能实现并发的读，因为它最终实现的是读写串行化，这样就大大降低了数据库的读写性能。加锁访问其实就是和MVCC相对的LBCC，即基于锁的并发控制（Lock-Based Concurrent Control），是四种隔离级别中级别最高的Serialize隔离级别。为了提出比LBCC更优越的并发性能方法，MVCC便应运而生。

它的最大好处便是，读不加锁，读写不冲突。
在MVCC中，读操作可以分成两类，
快照读（Snapshot read）和当前读（current read）。

在MySQL InnoDB中，简单的select操作，如 select * from table where ? 都属于快照读；

属于当前读的包含以下操作：
select * from table where ? lock in share mode; （加S锁）
select * from table where ? for update; （加X锁，下同）
insert, update, delete操作


MVCC是通过在每行纪录后面保存两个隐藏的列来实现的。这两个列，一个保存了行的创建时间，一个保存了行的过期时间（或删除时间），当然存储的并不是实际的时间值，而是系统版本号。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行纪录的版本号进行比较。

因为不知道当前读是否已经commit

因此我们需要知道当前活跃的事务有哪些


![avatar](http://www.xiatianhao.com/media/article_images/2019/06/26/tzxwqs.png)

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/25/cxintk.png)

```

在上面的例子中
查询时，生成Read-view
可能的情况一:[123,234]     读到5， 因为事务900已提交
可能的情况二:[123,234,900] 读到4， 因为知晓事务900尚未提交
```

#### Read-View

ReadView是判断一个事务开始一致性读的结构体，判断哪些数据可以被读到。

```
    dulint    low_limit_id;    /* 事务号 >= low_limit_id的记录，对于当前Read View都是不可见的 */

    dulint    up_limit_id;    /* 事务号 < up_limit_id ，对于当前Read View都是可见的 */

    ulint    n_trx_ids;    /* Number of cells in the trx_ids array */

    dulint*    trx_ids;    /* Additional trx ids which the read should

                not see: typically, these are the active

                transactions at the time when the read is

                serialized, except the reading transaction

                itself; the trx ids in this array are in a

                descending order */

dulint    creator_trx_id;    /* trx id of creating transaction, or

                (0, 0) used in purge */
```

#### 具体操作

(1)   select操作：

对于select的操作，只有同时满足如下2个条件才返回行记录

a)        行的修改版本号小于等于该事务版本号

b)        行的删除版本号要么没有被定义，要么大于事务版本号。

如果行的修改或者删除版本号大于事务号，说明行是被修改事务后启动的事务修改或者删除的。在可重复读的隔离级别下，后开始的事务堆数据的影响不应该被先开始的事务看见，所以应该忽略后开始的事务的更新或者删除操作。

(2)   insert操作：

新插入的行，行的修改版本号为更新为该事务的事务号。

(3)   update操作：

更新行的时候，InnoDB会把原来行复制一份，并把当前的事务号作为改行的修改版本号。

(4)   delete操作：

对于删除，InnoDB直接把改行的删除版本号修改为当前事务号，相当于标记为删除，而不是物理的删除。真是的删除是在InnoDB的purge线程去做的。


#### 出现不可重复读

第一次读活跃事务：[123,234,900]  读到4
中间事务900commit;
第二次读活跃事务：[123,234]  读到5

### 可重复读 级别 如何解决 不可重复读

第一次读活跃事务：[123,234,900]  读到4
中间事务900commit;
第二次读活跃事务：[123,234,900]  读到4

与 读已提交 隔离级别不同，可重复读隔离级下，只在第一次读时获取活跃事务，直到事务完成，都会一直复用该活跃事务，不再重新获取

如果有新事务在事务900之后又提交了新的update呢？

如事务1200修改v的值为6，并提交了事务，此时活跃事务仍是[123,234,900] ，但是1200已经结束，且不存在于活跃事务中，
1200的修改值6能代替4被读到吗？

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/25/2.png)

肯定不能，如果读到6，就跟前两次读到的4不同，又出现了"不可重复读"的问题

因此，只是判断是否在活跃事务中是不够的

还需要进行以下判断

如果trx_id< trx_id_min的话，那么表明该行记录所在的事务已经在本次新事务创建之前就提交了，所以该行记录的当前值是可见的。
如果trx_id>trx_id_max的话，那么表明该行记录所在的事务在本次新事务创建之后才开启，所以该行记录的当前值不可见。
如果trx_id_min <= trx_id <= trx_id_max, 那么表明该行记录所在事务在本次新事务创建的时候处于活动状态，从trx_id_min到trx_id_max进行遍历，如果trx_id等于他们之中的某个事务id的话，那么不可见。
从该行记录的DB_ROLL_PTR指针所指向的回滚段中取出最新的undo-log的版本号的数据，将该可见行的值返回。
需要注意的是，新建事务(当前事务)与正在内存中commit 的事务不在活跃事务链表中。

![avatar](http://www.xiatianhao.com/media/article_images/2019/06/25/csabwp.png)

例子中
活跃事务最小值 123
活跃事务最大值 900

1200大于最大值900，是当前事务开始后其他事务进行的修改，所以判断流程如下

V=6
trx_id 1200

事务id大于活跃事务最大值900，不读

v=5
trx_id 900 在活跃事务中，不读

v=4
trx_id 600 123~900之间， 且不在活跃事务中，读取

最后仍然读到4，保证了可重复读


### 可重复读 级别 是否解决 了幻读

![avatar](http://www.xiatianhao.com/media/article_images/2019/07/02/eyudkf.png)

```
read-only 型事务因为是MVCC快照读,所以可以根据read-view来过滤其他后起事务插入的数据
read-write 型事务会有修改写入操作,而修改写入是当前读
```
