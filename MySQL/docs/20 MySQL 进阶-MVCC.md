# 1 什么是 MVCC
MVCC，全称 Multi-Version Concurrency Control（多版本并发控制）。<br />MySQL 的其中两种隔离级别：

- READ COMMITTED：解决了脏写、脏读问题。
- REPEATABLE READ：解决了脏写、脏读、不可重复读、部分的幻读问题。

我们知道脏读、不可重复读、幻读都是由于读、写冲突产生的问题，在这两种隔离级别下，MySQL 使用 MVCC 来进行读操作，从而解决这些问题。<br />简单来说，MVCC 的思想是：当前事务读取一条记录时，如果有其他未提交的事务修改了这条记录，那么就利用 undo 日志，读取记录的历史版本，这样的话，读、写操作访问的是记录的不同版本，那么就不会产生读、写冲突。<br />读操作使用 MVCC 的方式，也称作一致性读、快照读。
# 2 版本链
版本链依赖于 undo 日志。<br />我们知道一条记录中，还包含 row_id（可能没有）、trx_id、roll_pointer 这 3 个隐藏列：

- trx_id：每次一个事务改动一条记录时，都会把该事务的事务 id 赋值给该记录 trx_id 隐藏列。
- roll_pointer：每对一条记录做一次改动，就会添加一条 undo 日志，记录与所有 undo 日志之间会通过 roll_pointer 属性链接成一个链表，称作“版本链”，如下所示：

![image.png](<../images/20 MySQL 进阶-MVCC/1.png>)<br />通过版本链我们就可以获取记录的历史版本。
# 3 ReadView
事务在读取记录之前，会生成一个 ReadView，对于 READ COMMITTED、REPEATABLE READ 隔离级别，生成 ReadView 的时机不同：

- READ COMMITTED：每次读取数据前都生成一个 ReadView。
- REPEATABLE READ：在第一次读取数据时生成一个 ReadView。

**简单来说，事务只会读取在它生成 ReadView 之前已提交事务所做的更改、或者自己修改过的记录。**<br />那么也就是说，如果在事务 A 读取记录时，一个未提交的事务 B 修改了记录，那么事务 A 就会去读取记录的历史版本，这样的话读、写操作访问的是记录的不同版本，避免了读、写冲突。<br />那么接下来要解决的问题，就是要在版本链中找到当前事务中可读的版本，也就是找到一条最新的、并且该版本对应的事务已提交的记录。<br />ReadView 中包含以下几个内容：

- m_ids：生成 ReadView 时，当前系统中“未提交的读写事务”的事务 id 列表。
- min_trx_id：m_ids 中的最小值。
- max_trx_id：生成 ReadView 时，系统中应该分配给下一个事务的 id 值（事务 id 是递增分配的）。
- creator_trx_id：生成该 ReadView 的事务的 id。

有了这个 ReadView，这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本是否在当前事务中可读：

- 如果被访问版本的 trx_id 属性值与 ReadView 中的 creator_trx_id 值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。 
- 如果被访问版本的 trx_id 属性值小于 ReadView 中的 min_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 前已经提交，所以该版本可以被当前事务访问。 
- 如果被访问版本的 trx_id 属性值大于 ReadView 中的 max_trx_id 值，表明生成该版本的事务在当前事务生成 ReadView 后才开启，所以该版本不可以被当前事务访问。 
- 如果被访问版本的 trx_id 属性值在 ReadView 的 min_trx_id 和 max_trx_id 之间，那就需要判断一下 trx_id 属性值是不是在 m_ids 列表中，如果在，说明创建 ReadView 时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提交，该版本可以被访问。 

如果某个版本的记录对当前事务不可见的话，那就顺着版本链找到下一个版本的记录，继续按照上边的步骤判断可见性，依此类推，直到版本链中的最后一个版本。如果最后一个版本也不可见的话，那么就意味着该条记录对该事务完全不可见，查询结果就不包含该记录。
# 4 MVCC 如何解决脏读、不可重复读、幻读问题
根据前面的描述，我们已经知道，**事务只会读取在它生成 ReadView 之前已提交事务所做的更改、或者自己修改过的记录**，那么显然脏读问题就已经解决了。<br />而 REPEATABLE READ 与 READ COMMITTED 相比，还能解决不可重复读、部分的幻读问题，这是由于二者生成 ReadView 的时机不同导致的：

- READ COMMITTED：每次读取数据前都生成一个 ReadView。
   - 如果在事务 A 第一次执行查询之后，事务 B 执行了修改、插入、删除操作，然后事务 A 再次执行相同的查询，那么就会导致两次读取结果的不一致，因此 READ COMMITTED 不能解决不可重复读和幻读问题。
- REPEATABLE READ：在第一次读取数据时生成一个 ReadView。
   - 因此无论读取几次，读取的都是在生成这个 ReadView 之前提交的最新记录，也就是读取的结果是相同的，因此 REPEATABLE READ 能够解决不可重复读和幻读问题。

但是上面遗漏了另一种情况：事务还可以读取生成 ReadView 之前，自己修改过的记录。比如：
```sql
-- 事务 A
SELECT * FROM user WHERE id >= 1; -- 1. 没有结果

UPDATE user SET name="Jack" WHERE id=1; -- 4. 将记录的 trx_id 属性设置为了事务 A 的 id
SELECT * FROM user WHERE id >= 1; -- 5. 读到了事务 B 插入的记录，发生幻读问题

-- 事务 B
INSERT INTO user(id, name, age) VALUES(1, "Leo", 18); -- 2. 插入一条记录
COMMIT; -- 3. 提交事务
```
所以说 MVCC 并没有完全解决幻读的问题，但是这种情况一般不会发生，因为我们不会去更新一条在当前事务看来不存在的数据。
# 5 MVCC 的问题
现在有事务 A 和事务 B，它们都执行一个转账操作，使 id=1 的用户的余额 +5：
```sql
-- 事务 A
BEGIN; -- 1.
SELECT balance FROM account WHERE id=1; -- 2. 获取初始余额，得到 5
-- 3. 执行 +5，新的 balance 为 10
UPDATE account SET balance=10 WHERE id=1; -- 4. balance 更新为 10
COMMIT; -- 10.

-- 事务 B
BEGIN; -- 5.
SELECT balance FROM account WHERE id=1; -- 6. 获取初始余额，使用 MVCC，读取到历史版本，得到 5
-- 7. 执行 +5，新的 balance 为 10
UPDATE account SET balance=10 WHERE id=1; -- 8. balance 更新为 10
COMMIT; -- 9.
```
可以看到在第 5 步中，由于使用了 MVCC，读取了记录的历史版本，虽然没有发生脏写、脏读、不可重复读、幻读问题，但是结果并不是我们预期的。因此，当需要读取记录的最新版本时，读操作需要使用锁。
