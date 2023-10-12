# 1 事务的起源
假设一个银行转账的场景，创建如下的表，存储用户的余额信息：
```sql
CREATE TABLE account (
	id int PRIMARY KEY,
	name varchar(10),
	balance int
);
```
表中数据如下所示：<br />![image.png](<../images/18 MySQL 进阶-事务/1.png>)<br />现在假设 Leo 要向 Jack 转账 5 元，那么这个转账过程可以分为以下两步：

1. UPDATE account SET balance = balance - 5 WHERE id = 1;
2. UPDATE account SET balance = balance + 5 WHERE id = 2; 

但是，如果执行完第一条语句，服务器崩溃了，那么就导致 Leo 的钱被扣了，但是却没有成功转给 Jack。<br />类似这种情况，就需要使用“事务”了。
# 2 事务的性质

1. 原子性：
   1. 事务中的操作要么全部执行，要么全部不执行（通过回滚）。
   2. 比如前面银行转账的例子，原子性就是保证这两条语句要么全部执行，要么全部不执行，不可能出现只执行了一条的情况。
2. 一致性：
   1. 数据库在事务执行过程中始终保持数据的正确性和完整性。
   2. 比如前面银行转账的例子，转帐前 Leo 和 Jack 的总余额为 13 元，那么转账后无论成功或失败，总余额也应该是 13 元，但是由于只执行了一条语句，导致最后总余额只有 8 元，不满足一致性。
3. 隔离性：
   1. 并发执行的各个事务之间不能互相干扰。
   2. 隔离性在底层是通过 MVCC 和锁机制实现的。
4. 持久性：
   1. 一旦事务提交，事务对数据库的更改应当在磁盘上保留下来，不会因为任何故障而丢失。
   2. MySQL 中修改数据时，并不会立刻写入磁盘，而是先保存到内存中的 Buffer Pool 中，所以就可能出现数据写入到了 Buffer Pool 中，此时服务器崩溃，导致数据没能写入磁盘。
   3. MySQL 中可以通过日志文件、备份和恢复等手段来保证持久性。
# 3 事务的语法
## 3.1 开启事务
可以使用以下两种方式开启事务：
```sql
-- 1. WORK 是可选的
BEGIN [WORK];
-- 2.
START TRANSACTION;
```
不同的是，START TRANSACTION 后还可以接一个修饰符，修饰符分两类：

- 读写模式：
   - READ ONLY：标识当前事务是一个只读事务，即只能读取数据，而不能修改数据。其实它只是不允许修改那些其他事务也能访问到的表中的数据，对于临时表来说，由于它们只能在当前会话中可见，所以是可修改的。
   - READ WRITE：默认值，标识当前事务是一个读写事务，也就是既可以读取数据，也可以修改数据。
- WITH CONSISTENT SNAPSHOT：启动一致性读，见下文。

可以同时指定读写模式和 WITH CONSISTENT SNAPSHOT：
```sql
START TRANSACTION READ ONLY, WITH CONSISTENT SNAPSHOT;
START TRANSACTION READ WRITE, WITH CONSISTENT SNAPSHOT;
```
## 3.2 提交事务
使用如下语句：
```sql
COMMIT [WORK]; -- WORK 是可选的
```
对于前面银行转账的例子，使用事务来解决问题：
```sql
BEGIN;
UPDATE account SET balance = balance - 5 WHERE id = 1;
UPDATE account SET balance = balance + 5 WHERE id = 2;
COMMIT;
```
## 3.3 手动中止事务（回滚）
即撤销事务的操作对数据库的修改，使用如下语句：
```sql
ROLLBACK [WORK]; -- WORK 是可选的
```
需要注意，ROLLBACK 语句是我们程序员手动的去回滚事务时才去使用的，如果事务在执行过程中遇到了某些错误而无法继续执行的话，事务自身会自动的回滚。
## 3.4 事务自动提交
MySQL 中有一个系统变量 autocommit，通过它设置事务自动提交：
```sql
SET autocommit = ON; -- 开启事务自动提交，为默认值
SET autocommit = OFF; -- 关闭事务自动提交
```
在不使用 BEGIN 或 START TRANSACTION 显式地开启事务时，每一条语句都算是一个独立的事务，执行完后自动提交，要想关闭这个功能：

- 显式地使用 BEGIN 或 START TRANSACTION 开启事务，那么该事务不会自动提交。
- 将系统变量 autocommit 设为 OFF 以关闭自动提交。
## 3.5 隐式提交
当我们使用了 BEGIN 或 START TRANSACTION 开启事务，或者将 autocommit 设为 OFF，那么事务不会自动提交，但是如果我们执行了某些语句，也会导致前边的事务的提交，就像执行了 COMMIT 一样，这种情况称为“隐式提交”，比如：
```sql
BEGIN;
INSERT INTO account VALUES(3, "Jacky", 10);
-- 会导致前面的语句所属的事务提交，该 CREATE 语句不属于这个事务的一部分，就像执行了 COMMIT 一样
CREATE TABLE user(id int PRIMARY KEY, name varchar(10));
```
会导致隐式提交的语句包括：

- 定义或修改数据库对象的数据定义语言 DDL：比如使用 CREATE 、 ALTER 、 DROP 等语句去修改数据库、表、视图、存储过程等。
- 隐式使用或修改 mysql 数据库中的表：当我们使用 ALTER USER、CREATE USER、DROP USER、GRANT、RENAME USER、REVOKE、SET PASSWORD 等语句时也会隐式的提交前边语句所属于的事务。
- 事务控制或关于锁定的语句：
   - 当我们在一个事务还没提交或者回滚时就又使用 START TRANSACTION 或者 BEGIN 语句开启了另一个事务时，会隐式的提交上一个事务。
   - 或者当前的 autocommit 系统变量的值为 OFF ，我们手动把它调为 ON 时，也会隐式的提交前边语句所属的事务。
   - 或者使用 LOCK TABLES、UNLOCK TABLES 等关于锁定的语句也会隐式的提交前边语句所属的事务。
- 加载数据的语句：比如我们使用 LOAD DATA 语句来批量往数据库中导入数据时，也会隐式的提交前边语句所属的事务。
- 关于 MySQL 复制的一些语句：使用 START SLAVE、STOP SLAVE、RESET SLAVE、CHANGE MASTER TO 等语句时也会隐式的提交前边语句所属的事务。
- 其它的一些语句：使用 ANALYZE TABLE、CACHE INDEX、CHECK TABLE、FLUSH、LOAD INDEX INTO CACHE、OPTIMIZE TABLE、REPAIR TABLE、RESET 等语句也会隐式的提交前边语句所属的事务。
## 3.6 保存点
在事务中设置保存点，那么执行 ROLLBACK 时就可以回滚到指定的保存点，而不是回滚整个事务。<br />设置保存点：
```sql
SAVEPOINT 保存点名称;
```
回滚到指定保存点：
```sql
ROLLBACK [WORK] TO [SAVEPOINT] 保存点名称; -- WORK 和 SAVEPOINT 是可选的
```
删除保存点：
```sql
RELEASE SAVEPOINT 保存点名称;
```
# 4 事务的隔离级别
## 4.1 事务并发执行的问题
### 4.1.1 脏写（写、写冲突）
一个事务修改了另一个未提交事务修改过的数据。<br />![image.png](<../images/18 MySQL 进阶-事务/2.png>)<br />如上所示，Session A 和 Session B 各开启了一个事务，它们首先都修改了同一条记录，等 Session A 的事务提交后，Session B 的事务发生了回滚，假设在 Session B 的事务执行前，number=1 的这条记录的 name 为 “刘备”，则回滚操作相当于执行：
```sql
UPDATE hero SET name="刘备" WHERE number=1;
```
则导致 Session A 对数据库的修改也不复存在。Session B 的回滚本应当只撤销自身对数据库的更改，所以这显然是不正确的。
### 4.1.2 脏读（读、写冲突）
一个事务读到了另一个未提交事务修改过的数据。<br />![image.png](<../images/18 MySQL 进阶-事务/3.png>)<br />如上所示，Session A 和 Session B 各开启了一个事务，首先 Session A 的事务读取到了 Session B 的事务修改过的数据，然后 Session B 的事务回滚，那么 Session A 中的事务相当于读到了一个不存在的数据。
### 4.1.3 不可重复读（读、写冲突）
一个事务中，多次执行相同的查询，由于其他事务修改或删除了已有的记录，导致先后读取到的数据不一致。<br />![image.png](<../images/18 MySQL 进阶-事务/4.png>)<br />如上所示，Session A 和 Session B 各开启了一个事务，Session A 的事务中多次执行相同的查询，每次得到的结果都不相同。
### 4.1.4 幻读（读、写冲突）
一个事务中，多次执行相同的查询，由于其他事务插入了新的记录，导致先后读取到的数据不一致。<br />![image.png](<../images/18 MySQL 进阶-事务/5.png>)<br />如上所示，Session A 和 Session B 各开启了一个事务，Session A 的事务第二次进行相同的查询时，发现返回的记录变多了。
## 4.2 四种隔离级别
前面四种问题，按其严重性排序为：脏写 > 脏读 > 不可重复读 > 幻读。<br />MySQL 中有四种隔离级别：

- READ UNCOMMITTED：能解决“脏写”问题。
- READ COMMITTED：能解决“脏写”、“脏读”问题。
- REPEATABLE READ：**默认的隔离级别**。SQL 标准中定义为只能解决“脏写”、“脏读”、“不可重复读”的问题，但是在 MySQL 中，该隔离级别已经解决了部分的“幻读”问题。 
- SERIALIZABLE：任何问题都不会发生，所有事务串行执行。

这四种隔离级别性能由好到坏依次为：READ UNCOMMITTED、READ COMMITTED、REPEATABLE READ、SERIALIZABLE。<br />设置隔离级别：
```sql
SET [GLOBAL|SESSION] TRANSACTION ISOLATION LEVEL 隔离级别; 
```
也可以在配置文件中设置隔离级别：
```bash
[mysqld]
transaction-isolation=SERIALIZABLE
```
查看隔离级别：
```sql
-- MySQL 5.7.20 之前
SELECT @@tx_isolation;
-- MySQL 5.7.20 开始
SELECT @@transaction_isolation;
```
注意，更改隔离级别之前，必须关闭所有连接，否则不会生效。
