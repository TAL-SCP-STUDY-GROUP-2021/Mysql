## Mysql

### 1.锁
数据库使用锁来支持对共享资源进行并发访问，提供数据的完整性和一致性。此外，数据库事务的隔离性也是通过锁实现的。
#### 1.1 锁的分类
##### 1.1.1 按模式分
锁的模式有：读意向锁，写意向锁，读锁，写锁和自增锁(auto_inc)：

读锁，又称共享锁（Share locks，简称 S 锁），加了读锁的记录，所有的事务都可以读取，但是不能修改，并且可同时有多个事务对记录加读锁。

写锁，又称排他锁（Exclusive locks，简称 X 锁），或独占锁，对记录加了排他锁之后，只有拥有该锁的事务可以读取和修改，其他事务都不可以读取和修改，并且同一时间只能有一个事务加写锁。
```mysql
CREATE DATABASE test;
USE test;
CREATE TABLE goods (
    id INT AUTO_INCREMENT, 
    num INT NOT NULL DEFAULT '0', 
    PRIMARY KEY(id));
CREATE TABLE trade (
    id INT AUTO_INCREMENT, 
    goods_id INT, 
    user_id INT, 
    PRIMARY KEY(id));
SESSION A:
    LOCK TABLES goods WRITE;
    LOCK TABLES trade READ;
    UNLOCK TABLES;
SESSION B:
    SELECT * FROM goods;
    SELECT * FROM trade;
```
表锁和行锁虽然锁定范围不同，但是会相互冲突。要加表锁时，势必要先遍历该表的所有记录，判断是否加有排他锁。这种遍历检查的方式显然是一种低效的方式，MySQL 引入了意向锁，来检测表锁和行锁的冲突。

意向锁也是表级锁，也可分为读意向锁（IS 锁）和写意向锁（IX 锁）。当事务要在记录上加上读锁或写锁时，要首先在表上加上意向锁。这样判断表中是否有记录加锁就很简单了，只要看下表上是否有意向锁就行了。

意向锁之间是不会产生冲突的，也不和 AUTO_INC 表锁冲突，它只会阻塞表级读锁或表级写锁，另外，意向锁也不会和行锁冲突，行锁只会和行锁冲突。
```mysql
SESSION A:
    BEGIN;
    UPDATE goods SET num = 11 WHERE id=1;
SESSION B:
    BEGIN;
    UPDATE goods SET num = 11 WHERE id=1;
SESSION C:
    SELECT * FROM information_schema.INNODB_TRX\G
    SELECT * FROM information_schema.INNODB_LOCKS\G
    SELECT * FROM information_schema.INNODB_LOCK_WAITS\G
    SHOW status LIKE 'innodb_row_lock_%';
    SHOW OPEN TABLES where In_use > 0;
```
AUTO_INC锁又叫自增锁（一般简写成AI锁），是一种表锁，当表中有自增列（AUTO_INCREMENT）时出现。当插入表中有自增列时，数据库需要自动生成自增值，它会先为该表加 AUTO_INC 表锁，阻塞其他事务的插入操作，这样保证生成的自增值肯定是唯一的。AUTO_INC 锁具有如下特点：

AUTO_INC锁互不兼容，也就是说同一张表同时只允许有一个自增锁；
自增值一旦分配了就会 +1，如果事务回滚，自增值也不会减回去，所以自增值可能会出现中断的情况。
显然，AUTO_INC 表锁会导致并发插入的效率降低，为了提高插入的并发性，MySQL 从 5.1.22 版本开始，引入了一种可选的轻量级锁（mutex）机制来代替 AUTO_INC 锁，可以通过参数 innodb_autoinc_lock_mode 来灵活控制分配自增值时的并发策略。

兼容表

|-|IS|IX|S|X|AI|
|---|---|---|---|---|---|
|IS|&radic;|&radic;|&radic;|-|&radic;|
|IX|&radic;|&radic;|-|-|&radic;|
|S|&radic;|-|&radic;|-|-|
|X||||||
|AI|&radic;|&radic;||||
总结起来有下面几点：

意向锁之间互不冲突；
S 锁只和 S/IS 锁兼容，和其他锁都冲突；
X 锁和其他所有锁都冲突；
AI 锁只和意向锁兼容

##### 1.1.2 按粒度分
1). 表锁

表锁是指对一整张表加锁，一般是执行DDL时使用，由Mysql Server实现。

表锁使用的是一次性锁技术，也就是说，在会话开始的地方使用 lock 命令将后续需要用到的表都加上锁，在表释放前，只能访问这些加锁的表，不能访问其他表，直到最后通过 unlock tables 释放所有表锁。

除了使用 unlock tables 显示释放锁之外，会话持有其他表锁时执行lock table 语句会释放会话之前持有的锁；会话持有其他表锁时执行 start transaction 或者 begin 开启事务时，也会释放之前持有的锁。

```mysql
SESSION A:
    LOCK TABLES goods WRITE;
    SELECT * FROM trade;
    LOCK TABLES trade READ;
    UNLOCK TABLES;
SESSION B:
    SELECT * FROM goods;
    SELECT * FROM goods;
    SELECT * FROM trade;

```

2). 行锁

不同存储引擎的行锁实现不同，常用的引擎InnoDB支持行锁，MyISAM不支持行锁，后续都是针对InnoDB引擎。

在了解 InnoDB 的加锁原理前，需要对其存储结构有一定的了解。InnoDB 是聚簇索引，也就是 B+树的叶节点既存储了主键索引也存储了数据行。而 InnoDB 的二级索引的叶节点存储的则是主键值，所以通过二级索引查询数据时，还需要拿对应的主键去聚簇索引中再次进行查询。
因此使用主键索引需要加一把锁，使用二级索引需要在二级索引和主键索引上各加一把锁。

行锁根据作用的范围又可以分为Record Lock(记录锁)、Gap Lock(间隙锁)、Next-key Lock和II GAP(插入意向锁)。

记录锁是最简单的行锁，就是对一条记录加锁。

当 SQL 语句无法使用索引时，会进行全表扫描，这个时候 MySQL 会给整张表的所有数据行加记录锁，再由 MySQL Server层进行过滤。在MySQL Server层进行过滤的时候，如果发现不满足 WHERE 条件，会释放对应记录的锁。这样做，保证了最后只会持有满足条件记录上的锁，但是每条记录的加锁操作还是不能省略的。

所以更新操作必须要根据索引进行操作，没有索引时，不仅会消耗大量的锁资源，增加数据库的开销，还会极大的降低了数据库的并发性能。
```mysql
SELECT * FROM goods WHERE id > 0 LOCK IN SHARE MODE;
SELECT * FROM goods WHERE id > 0 FOR UPDATE;
```
间隙锁是一种加在两个索引之间的锁，或者加在第一个索引之前，或最后一个索引之后的间隙。这个间隙可以跨一个索引记录，多个索引记录，甚至是空的。使用间隙锁可以防止其他事务在这个范围内插入或修改记录，保证两次读取这个范围内的记录不会变，从而不会出现幻读现象。
举个例子，id=6这条记录不存在，更新这条数据还会加锁吗？答案是可能有，这取决于数据库的隔离级别。这种情况下，在 RC 隔离级别不会加任何锁，在 RR 隔离级别会在 id = 6 前后两个索引之间加上间隙锁。
```mysql
SESSION A:
    BEGIN;
    UPDATE goods SET num = 10 WHERE id=3;
SESSION B:
    INSERT goods (id, num) VALUES (2, 11);
```

值得注意的是，间隙锁和间隙锁之间是互不冲突的，间隙锁唯一的作用就是为了防止其他事务的插入，所以加间隙 S 锁和加间隙 X 锁没有任何区别。

Next-key锁是记录锁和间隙锁的组合，它指的是加在某条记录以及这条记录前面间隙上的锁。和间隙锁一样，在RC隔离级别下没有Next-key锁，只有RR隔离级别才有。
InnoDB中行锁默认使用算法Next-Key Lock，只有当查询的索引是唯一索引或主键时，InnoDB会对Next-Key Lock进行优化，将其降级为Record Lock，即仅锁住索引本身，而不是范围。当查询的索引为辅助索引时，InnoDB则会使用Next-Key Lock进行加锁。InnoDB对于辅助索引有特殊的处理，不仅会锁住辅助索引值所在的范围，还会将其下一键值加上Gap LOCK。
```mysql
SESSION A:
    BEGIN;
    SELECT * FROM goods WHERE price = 6 FOR UPDATE;
SESSION B:
    select * from goods where id=9 for update;
    insert goods (num,price) values (10,3);
    insert goods (num,price) values (10,8);
```
|-|RECORD|GAP|NEXT-KEY|
|---|---|---|---|
|RECORD|-|&radic;|-|
|GAP|&radic;|&radic;|&radic;|
|NEXT-KEY|-|&radic;|-|
### 2.事务
#### 2.1 事务是什么
数据库的事务（Transaction）是一种机制、一个操作序列，包含了一组数据库操作命令。事务把所有的命令作为一个整体一起向系统提交或撤销操作请求，即这一组数据库命令要么都执行，要么都不执行，因此事务是一个不可分割的工作逻辑单元。

在数据库系统上执行并发操作时，事务是作为最小的控制单元来使用的，特别适用于多用户同时操作的数据库系统。例如，航空公司的订票系统、银行、保险公司以及证券交易系统等。
#### 2.2 事务的特性
事务具有 4 个特性，即原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability），这 4 个特性通常简称为 ACID。
###### 原子性
事务是一个完整的操作。事务的各元素是不可分的（原子的）。事务中的所有元素必须作为一个整体提交或回滚。如果事务中的任何元素失败，则整个事务将失败。
###### 一致性
当事务完成时，数据必须处于一致状态。也就是说，在事务开始之前，数据库中存储的数据处于一致状态。在正在进行的事务中. 数据可能处于不一致的状态，如数据可能有部分被修改。然而，当事务成功完成时，数据必须再次回到已知的一致状态。通过事务对数据所做的修改不能损坏数据，或者说事务不能使数据存储处于不稳定的状态。
###### 隔离性
对数据进行修改的所有并发事务是彼此隔离的，这表明事务必须是独立的，它不应以任何方式依赖于或影响其他事务。修改数据的事务可以在另一个使用相同数据的事务开始之前访问这些数据，或者在另一个使用相同数据的事务结束之后访问这些数据。
###### 持久性
事务的持久性指不管系统是否发生了故障，事务处理的结果都是永久的。

一个事务成功完成之后，它对数据库所作的改变是永久性的，即使系统出现故障也是如此。也就是说，一旦事务被提交，事务对数据所做的任何变动都会被永久地保留在数据库中。

Mysql中事务的隔离性由多版本控制机制和锁实现，而原子性、一致性和持久性通过InnoDB的redo log、undo log和Force Log at Commit机制来实现。

#### 2.3 Mysql中对事务的操作

|语句|含义|
|---|---|
|BEGIN; START TRANSACTION;SET AUTOCOMMIT=0|开启事务|
|COMMIT; COMMIT WORK|提交事务|
|ROLLBACK; ROLLBACK WORK|回滚事务|
|SAVEPOINT identifier|在事务中创建一个保存点|
|RELEASE SAVEPOINT identifier|删除事务中的保存点|
|ROLLBACK TO identifier|回滚到指定保存点|
```mysql
SET [GLOBAL || SESSION] TRANSACTION ISOLATION LEVEL [];
```
用户可以通过
```mysql 
COMPLETION_TYPE
```
参数来控制事务结束后的动作。

|COMPLETION_TYPE|动作|
|---|---|
|0|无|
|1|马上开启一个隔离级别相同的事务|
|2|COMMIT AND RELEASE,事务结束后断开与服务器连接|
```mysql
CREATE DATABASE student;
CREATE TABLE `stu_info` (
`stu_id` int NOT NULL AUTO_INCREMENT,
`name` varchar(25) DEFAULT NULL,
`age` int DEFAULT NULL,
 PRIMARY KEY (`stu_id`)
) ENGINE=InnoDB AUTO_INCREMENT=0;
INSERT stu_info (name,age) VALUES ('ldy',25);


SELECT * FROM stu_info;
SET @@COMPLETION_TYPE = 0;
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
COMMIT;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
ROLLBACK;
SELECT * FROM stu_info;

SELECT * FROM stu_info;
SET @@COMPLETION_TYPE = 1;
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
COMMIT;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
ROLLBACK;
SELECT * FROM stu_info;

SELECT * FROM stu_info;
SET @@COMPLETION_TYPE = 2;
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
COMMIT;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
ROLLBACK;
SELECT * FROM stu_info;
```
#### 2.4 事务并发带来的问题
1、脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
```mysql
SELECT @@TRANSACTION_ISOLATION;
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

session A:
BEGIN;
SELECT * FROM stu_info;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
commit;

session B:
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;

```
2、不可重复读：事务A多次读取同一数据，事务B在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
```mysql
SELECT @@TRANSACTION_ISOLATION;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

session A:
BEGIN;
SELECT * FROM stu_info;
wait B;
SELECT * FROM stu_info;
commit;

session B:
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
COMMIT;

```

3、幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。
```mysql
SELECT @@TRANSACTION_ISOLATION;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

session A:
BEGIN;
SELECT * FROM stu_info;
wait B;
SELECT * FROM stu_info;
commit;

session B:
BEGIN;
UPDATE stu_info SET age = age+1 WHERE stu_id = 1;
INSERT INTO stu_info (name, age) VALUES ('ldy', 22);
COMMIT;
```
#### 2.5 Mysql的隔离级别

|隔离级别|脏读|不可重复读|幻读|
|---|---|---|---|
|READ UNCOMMITTED|&times;|&times;|&times;|
|READ COMMITTED|&radic;|&times;|&times;|
|REPEATABLE READ|&radic;|&radic;|&times;|
|SERIALIZE|&radic;|&radic;|&radic;|

#### 2.6 InnoDB的MVCC(多版本并发控制)机制
MVCC多版本并发控制(Multi-Version Concurrency Control)是MySQL中基于乐观锁理论实现隔离级别的方式，用于实现读已提交和可重复读取隔离级别。

InnoDB行记录有三个隐藏字段：分别对应该行的rowid、事务号db_trx_id和回滚指针db_roll_ptr，其中db_trx_id表示最近修改的事务的id，db_roll_ptr指向回滚段中的undo log。
当事务2使用UPDATE语句修改该行数据时，会首先使用排他锁锁定改行，将该行当前的值复制到undo log中，然后再真正地修改当前行的值，最后填写事务ID，使用回滚指针指向undo log中修改前的行。

REPEATABLE READ隔离级别下事务开始后使用MVVC机制进行读取时，会将当时活动的事务id记录下来，记录到Read View中。READ COMMITTED隔离级别下则是每次读取时都创建一个新的Read View。
Read View是InnoDB中用于判断记录可见性的数据结构，记录了一些用于判断可见性的属性。
creator_trx_id: 当前事务的 id;

m_ids: 当前系统中所有的活跃事务的 id，活跃事务指的是当前系统中开启了事务，但是还没有提交的事务;

min_trx_id: 当前系统中，所有活跃事务中事务 id 最小的那个事务，也就是 m_id 数组中最小的事务 id;

max_trx_id: 当前系统中事务的 id 值最大的那个事务 id 值再加 1，也就是系统中下一个要生成的事务 id。

如果当前数据的 row_trx_id 小于 min_trx_id，那么表示这条数据是在当前事务开启之前，其他的事务就已经将该条数据修改了并提交了事务(事务的 id 值是递增的)，所以当前事务能读取到。

如果当前数据的 row_trx_id 大于等于 max_trx_id，那么表示在当前事务开启以后，过了一段时间，系统中有新的事务开启了，并且新的事务修改了这行数据的值并提交了事务，所以当前事务肯定是不能读取到的，因此这是后面的事务修改提交的数据.

如果当前数据的 row_trx_id 处于 min_trx_id 和 max_trx_id 的范围之间，又需要分两种情况：
row_trx_id 在 m_ids 数组中，那么当前事务不能读取到。row_trx_id 在 m_ids 数组中表示的是和当前事务在同一时刻开启的事务，修改了数据的值，并提交了事务，所以不能让当前事务读取到;
row_trx_id 不在 m_ids 数组中，那么当前事务能读取到。row_trx_id 不在 m_ids 数组中表示的是在当前事务开启之前，其他事务将数据修改后就已经提交了事务，所以当前事务能读取到。

#### 2.7 RR隔离级别下Mysql如何解决幻读
通过MVCC机制，虽然让数据变得可重复读，但用户读到的数据可能是历史数据，不是数据库最新的数据。这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库最新版本数据的方式，叫当前读 (current read)。

在快照读情况下，Mysql通过MVCC来避免幻读, SELECT使用快照读，而UPDATE,DELETE,INSERT使用当前读。

在当前读情况下，Mysql通过X锁或Next-key来避免其他事务修改:

1) 使用串行化读的隔离级别

2) (update、delete)当where条件为主键时，通过对主键索引加record locks(索引加锁/行锁)处理幻读。

3) (update、delete)当where条件为非主键索引时，通过next-key锁处理。next-key是record locks(索引加锁/行锁) 和 gap locks(间隙锁，每次锁住的不光是需要使用的数据，还会锁住这些数据附近的数据)的结合。
