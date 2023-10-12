# 1 为什么使用锁
事务并发问题有四种：脏写、脏读、不可重复读、幻读。

- 对于脏写问题，解决方法就是对写操作加独占锁（写操作 INSERT、UPDATE、DELETE 是默认加独占锁的，不需要我们自己加），这样的话，当事务 A 修改记录时，事务 B 尝试修改记录就会阻塞，当事务 A 提交后，事务 B 才能修改。
- 对于脏读、不可重复读、幻读问题，在“MySQL 进阶-MVCC”一章中，已经讲了可以通过 MVCC 来解决，但是由于 MVCC 可能读取的是记录的历史版本，所以仍可能出问题，如果我们需要读取记录的最新版本，这时就需要使用锁了。

用锁来解决“MySQL 进阶-MVCC”一章中提到的问题：
```sql
-- 事务 A
BEGIN; -- 1.
SELECT balance FROM account WHERE id=1 FOR UPDATE; -- 2. 加独占锁，获取初始余额，得到 5
-- 3. 执行 +5，新的 balance 为 10
UPDATE account SET balance=10 WHERE id=1; -- 4. 将 balance 更新为 10
COMMIT; -- 7. 事务提交，释放独占锁

-- 事务 B
BEGIN; -- 5.
-- 6. 阻塞，需要等事务 A 的独占锁释放
-- 8. 事务 A 提交后，继续执行，此时可以读到事务 A 修改过的结果，读取 balance，得到 10
SELECT balance FROM account WHERE id=1 FOR UPDATE;
-- 9. 执行 +5，新的 balance 为 15
UPDATE account SET balance=10 WHERE id=1; -- 10. balance 更新为 15
COMMIT; -- 11.
```
锁解决脏读、不可重复读、幻读问题的方式是，对读、写操作都加锁。<br />读操作使用锁的方式也称作锁定读、当前读。
# 2 锁结构与索引
注意，这里指的是行级锁。<br />锁是内存中的一个结构，它是跟索引中的一条记录相关联的，如下所示：<br />![image.png](<../images/21 MySQL 进阶-锁/1.png>)<br />锁结构中比较重要的两个信息：

- trx：代表这个锁结构是哪个事务生成的。
- is_waiting：代表当前事务是否在等待。

当一个事务尝试加锁时，便会生成一个锁结构与索引中的相应记录关联，加锁成功则将 is_waiting 设为 false，加锁失败则将 is_waiting 设为 true 并等待锁释放。<br />注意：**如果查询未命中任何索引，即为全表扫描时，行级锁将变为表级锁**。
# 3 加锁的语法
## 3.1 行级锁
行级锁用于对记录进行加锁，从互斥性的维度，可以分为两种：

1. 共享锁（S 锁）：如果事务 A 对某行记录加 S 锁，则其他事务可以再对该记录加 S 锁，但加 X 锁将被阻塞，直到事务 A 提交后将锁释放。
2. 独占锁（X 锁）：如果事务 A 对某行记录加 X 锁，则其他事务对该记录加 S 锁和 X 锁时都将被阻塞，直到事务 A 提交后将锁释放。

语法如下：
```sql
SELECT * FROM `user` WHERE id=1 LOCK IN SHARE MODE; -- 共享锁
SELECT * FROM `user` WHERE id=1 FOR UPDATE; -- 独占锁
```
另外，DELETE、INSERT、UPDATE 语句会自动加上独占锁。<br />行级锁会在事务提交、回滚、或会话断开时被释放。
## 3.2 表级锁
表级锁用于对整个表进行加锁，同样分为共享锁和独占锁两种。<br />语法如下：
```sql
LOCK TABLES `user` READ; -- 对 user 表加共享锁
LOCK TABLES `user` WRITE; -- 对 user 表加独占锁

UNLOCK TABLES; -- 释放锁
```
另外，当会话断开时，也会释放表级锁。
## 3.3 全局锁
全局锁会对整个数据库进行加锁，加锁后，整个数据库变为只读状态。<br />全局锁主要在对整库进行备份时使用，这样在备份数据库期间，不会因为数据或表结构的更新，而出现备份文件的数据与预期的不一样。
```sql
FLUSH TABLES WITH READ LOCK; -- 加全局锁，同时将缓存中的数据刷新到磁盘中
UNLOCK TABLES;-- 释放全局锁
```
# 4 行级锁的实现
## 4.1 记录锁（Record Lock）
在 READ COMMITTED 隔离级别下，行级锁为记录锁。<br />记录锁就是仅仅把一条记录锁上，能解决脏读和不可重复读的问题。<br />如下所示，对 number=8 的记录加上一个记录锁：<br />![image.png](<../images/21 MySQL 进阶-锁/2.png>)<br />如果是 S 锁，那么其他事务获取该条记录的 X 锁时，将会阻塞；如果是 X 锁，那么其他事务获取该条记录的 S 锁或 X 锁时，都将会阻塞。<br />注意：如果其他事务插入一条 number=8 的记录，不会阻塞，因为根本不是对一条记录加的锁。
## 4.2 间隙锁（Gap Lock）
间隙锁只在 REPEATABLE READ 隔离级别下生效。<br />间隙锁是为了解决幻读问题而产生的，它是针对插入操作，锁住的是一个间隙。<br />间隙指的是一条记录与它前面一条记录之间的间隙，间隙锁将会锁住整个间隙，如下所示，对 number=8 的记录加上一个间隙锁：<br />![image.png](<../images/21 MySQL 进阶-锁/3.png>)<br />当其他事务尝试插入一条记录时，它会在索引中定位到这条新记录的下一条记录，如果这条记录上有间隙锁，那么插入操作将会阻塞。<br />网上都说间隙锁锁住的区间是开区间，但经测试应该是左闭右开区间，比如这里在 number=8 的记录上加间隙锁，那么锁住的区间是 number 列值 [3, 8) 的区间，也就是说，如果插入一条 number=3 的记录，也会阻塞。
## 4.3 临键锁（Next-key Lock）
临键锁只在 REPEATABLE READ 隔离级别下生效。<br />临键锁是记录锁+间隙锁的组合，也就是说，针对 SELECT（如果锁的互斥性冲突的话）、UPDATE、DELETE 操作，锁住记录自身；针对 INSERT 操作，锁住它前面的间隙。<br />如下所示，对 number=8 的记录加上一个间隙锁：<br />![image.png](<../images/21 MySQL 进阶-锁/4.png>)<br />那么当其他事务尝试获取 number=8 的记录的冲突的锁时，将会阻塞；当其他事务尝试插入 number 值在 [3, 8) 内的新记录时，将会阻塞。
## 4.4 一条 SQL 语句会加哪些锁
在 REPEATABLE READ 隔离级别下，当执行一条加锁的 SQL 语句时，并不是只使用一个锁，而是会使用多个锁：
```sql
CREATE TABLE `user` (
  `id` int(11),
  `name` varchar(10),
  `age` int(11),
  PRIMARY KEY (`id`),
  KEY `a` (`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

INSERT INTO `user` VALUES(1, "a", 1);
INSERT INTO `user` VALUES(2, "b", 5);
INSERT INTO `user` VALUES(3, "c", 10);

-- 事务 A
BEGIN;
SELECT * FROM `user` WHERE age > 3 AND age < 8 LOCK IN SHARE MODE;

-- 事务 B
INSERT INTO `user` VALUES(4, "d", 0); -- 不阻塞
INSERT INTO `user` VALUES(5, "e", 1); -- 阻塞
SELECT * FROM `user` WHERE age=5 FOR UPDATE; -- 阻塞
INSERT INTO `user` VALUES(6, "f", 9); -- 阻塞
INSERT INTO `user` VALUES(7, "g", 10); -- 不阻塞
```
根据上述结果分析，事务 A 在 age=5 的记录上加了临键锁、在 age=10 的记录上加了间隙锁：

- 针对 SELECT（如果锁的互斥性冲突的话）、UPDATE、DELETE 操作，锁住 age=5 的记录；
- 针对 INSERT 操作，锁住 age 列值为 [1, 10) 的区间。
# 5 其他表级锁
除了前面讲的使用 SQL 语句显式地加表级锁外，还有其他的表级锁，但是这些表级锁不需要我们自己手动去获取，所以用不上。
## 5.1 元数据锁
元数据锁锁住的是表结构，当我们向一张表创建一个索引、修改一个字段的名称等等，也就是说执行 DDL 语句时，就会加上元数据锁。<br />当表加上元数据锁后，其他事务做任何操作，比如执行 SELECT、INSERT、DELETE、UPDATE 语句都会阻塞；反过来，当其他事务在做这些操作时，给表加元数据锁（执行 DDL 语句）也会阻塞。
## 5.2 意向锁
意向锁分为意向共享锁（IS 锁）和意向独占锁（IX 锁）。

- 当对表中的记录加了行级共享锁时，那么就会同时对表加上一个意向共享锁。意向共享锁与表级独占锁冲突。
- 当对表中的记录加了行级独占锁时，那么就会同时对表加上一个意向独占锁。意向独占锁与表级共享锁、表级独占锁冲突。

我们知道，表级锁跟行级锁是存在冲突的：表级共享锁与行级独占锁冲突，表记独占锁与行级共享锁、行级独占锁冲突。<br />当没有意向锁时，比如，如果有其他事务尝试获取表级锁，就需要遍历所有的记录，查看是否有哪条记录加了与之冲突的行级锁，判断当前事务是否要阻塞，这样的效率太低了。<br />但是有了意向锁之后，当事务尝试获取表级锁时，只要查看表上是否加了与之冲突的意向锁，就知道当前事务是否需要阻塞了。
## 5.3 自增锁
用于保证自增列值的唯一性。<br />通常我们会对主键列设置自增（AUTO_INCREMENT），意味着我们不需要自己设置这一列的值，MySQL 会自动分配。<br />但是在并发的情况下，比如同时有两个事务进行插入，这时会使用自增锁保证 MySQL 分配的自增列值不会重复。<br />自增锁分为两种：

- AUTO-INC 锁：每次执行插入语句前都会在表级别加上一个 AUTO-INC 锁， 然后为自增列分配递增的值，在该语句执行结束后，再把 AUTO-INC 锁释放掉。事务在持有 AUTO-INC 锁的过程中，其他事务的插入语句都要被阻塞，性能较低。 
- 轻量级锁：在为插入语句生成自增列的值时获取一下这个轻量级锁，然后生成本次插入语句需要用到的自增列的值之后，就把该轻量级锁释放掉，并不需要等到整个插入语句执行完才释放锁。 

可以通过系统变量 innodb_autoinc_lock_mode 设置使用哪种自增锁：

- innodb_autoinc_lock_mode = 0：使用 AUTO-INC 锁。
- innodb_autoinc_lock_mode = 1：默认值。
   - 如果我们的插入语句在执行前不可以确定具体要插入多少条记录（无法预计即将插入记录的数量），比方说使用 INSERT ... SELECT、REPLACE ... SELECT 或者 LOAD DATA 这种插入语句，这时将使用 AUTO-INC 锁。
   - 如果我们的插入语句在执行前就可以确定具体要插入多少条记录，比如一个普通的 INSERT 语句，这时将使用轻量级锁。
- innodb_autoinc_lock_mode = 2：使用轻量级锁。
# 6 乐观锁
乐观锁不是真正的锁，它针对更新操作，通过在表中增加一个版本号字段，每次更新一条记录时，将该记录的版本号+1。事务开始时首先获取要更新的记录的版本号，然后在更新时检测版本号是否发生改变，如果改变了，说明记录已经被其他事务修改过，则放弃更新。
```sql
BEGIN;
SELECT version FROM table WHERE id = 1; -- 假设得到的 version 为 19
-- 其他操作，比如我们将 age 列的值 +10，得到的结果为 30
UPDATE TABLE SET age=30 WHERE id=1 AND version=19 -- 如果 version 不等于 19，说明该记录被其他事务修改过
```
除了使用版本号外，也可以使用时间戳，每次更新记录时将时间戳更新。
