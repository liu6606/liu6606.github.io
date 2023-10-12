# 1 避免使用 SELECT *
只查询我们真正需要的列，而不是使用 SELECT * 查询所有的列，原因有：

1. 它查询出了不必要的列，会增加网络传输的时间。
2. 它可能会导致大量的回表，降低查询的性能。
# 2 用 UNION ALL 替代 UNION
```sql
SELECT id, name FROM user
UNION
SELECT id, name FROM student;
```
UNION 可以排除重复的记录，而排除重复这个过程需要遍历、排序、比较所有记录，更加耗时。因此除非要求一定要排除重复的记录，那么使用 UNION ALL 而不是 UNION。
# 3 小表驱动大表
即让小表作为驱动表，大表作为被驱动表。<br />在 JOIN 查询中存在驱动表和被驱动表的概念，其工作原理是这样的：遍历驱动表中的每一条记录，在被驱动表中查找匹配的记录。<br />“遍历驱动表中的每一条记录”，这是一个全表扫描操作，时间复杂度为 O(n)；而“在被驱动表中查找匹配的记录“，这是一个查询操作，可以走索引，时间复杂度为 O(log n)。<br />当执行下面的连接查询时：
```sql
SELECT t1.id, t1.name FROM t1 INNER JOIN t2 ON t1.id=t2.id;
```
假设 t1 为大表，有 1000 条数据；t2 为小表，有 10 条数据。那么：

1. 当 t1 作为驱动表时，花费的时间为 1000*log10=1000。
2. 当 t2 作为驱动表时，花费的时间为 10*log1000=30。

所以**当小表作为驱动表时，速度更快，因为让大表走索引能节省更多时间**。<br />LEFT JOIN 会使用左侧的表作为驱动表；RIGHT JOIN 会使用右侧的表作为驱动表；而对于 INNER JOIN，MySQL 会自动选择合适的表作为驱动表，也可以使用 STRAIGHT_JOIN 替代 INNER JOIN 来指定左侧的表为驱动表。<br />另外，在子查询中也存在驱动表和被驱动表，详见下文。
# 4 子查询善用 EXISTS 和 IN
以下是一个 IN 的例子：
```sql
SELECT id FROM t1 WHERE id IN(SELECT id FROM t2);
```
与之等价的 EXISTS 的写法：
```sql
SELECT id FROM t1 WHERE EXISTS(SELECT 1 FROM t2 WHERE t1.id=t2.id);
```
EXISTS 的工作原理是，如果子查询的结果集非空，那么返回 true，否则返回 false。<br />二者的区别在于，IN 将子查询的表作为驱动表，EXISTS 将主查询的表作为驱动表。<br />根据小表驱动大表的原则：如果子查询的表是小表，那么使用 IN；如果子查询的表是大表，那么使用 EXISTS。<br />另外，对于 NOT EXISTS，无论何种情况，NOT EXISTS 都比 NOT IN 更快，因为使用 NOT IN 时（反向查询），被驱动表（主查询的表）索引失效，而使用 NOT EXISTS 时，被驱动表（子查询的表）能正常使用索引。
# 5 批量插入
当要插入多条数据时，最好一次性批量插入，而不是多次插入，因为客户端每次向 MySQL 服务器发送一条请求都是要消耗性能的。<br />比如：
```sql
INSERT INTO user (id, name, age) VALUES(1, "Leo", 18);
INSERT INTO user (id, name, age) VALUES(2, "Jack", 19);
INSERT INTO user (id, name, age) VALUES(3, "Alice", 20);
```
优化为：
```sql
INSERT INTO user (id, name, age) VALUES(1, "Leo", 18), (2, "Jack", 19), (3, "Alice", 20);
```
另外，批量插入的数据也不宜过多，否则 MySQL 需要较长时间才能返回结果。
# 6 巧用 LIMIT
在深分页的情况下，比如上一次分页已经查询到了第 100000 条数据，那么下一次分页：
```sql
SELECT id, name FROM user LIMIT 100000, 20;
```
这种情况下，会通过全表扫描，查询 100020 条数据，然后抛弃前面 100000 条数据，耗时很长。<br />我们可以记录每次分页查询出的最大 id，假设上一次分页查询出的最大 id 为 139999，那么下一次分页时，就可以优化为：
```sql
SELECT id, name FROM user WHERE id>139999 LIMIT 20;
```
此时可以使用索引，快速地跳过前 100000 条数据，需要注意这种方法要求 id 是自增的。<br />另外，如果我们能明确要查询的数据只有一条，可以加上 LIMIT 1，那么查询到我们想要的数据后就可以停止查询了。
# 7 使用连接查询代替子查询
子查询：
```sql
SELECT id FROM t1 WHERE id IN(SELECT id FROM t2 WHERE id>10);
```
使用连接查询代替：
```sql
SELECT t1.id FROM t1 INNER JOIN t2 ON t1.id=t2.id WHERE t2.id>10;
```
子查询的缺点在于：

1. 执行子查询会生成临时表，查询完毕后，需要将临时表删除，这会带来性能消耗。
2. MySQL 查询优化器对子查询的优化能力较弱。
# 8 JOIN 的表不应过多
根据阿里巴巴开发者手册的规定，JOIN 表的数量不应该超过 3 个。<br />原因如下：

1. JOIN 的表过多时，执行时需要进行大量的数据匹配和比较操作，性能消耗较大。
2. 过多的 JOIN 会使得可读性和可维护性变差，容易出现错误。
# 9 ORDER BY 优化
如果 ORDER BY 排序的列属于同一个联合索引，ORDER BY 为升序排序，并且列的顺序与联合索引列的顺序相同，那么此时可以避免额外的排序，因为从索引中查询出的记录直接就是按照 ORDER BY 的顺序排序好的。比如：
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name_age` (`name`,`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

SELECT name FROM user ORDER BY name, age;
```
其它情况下，都需要对结果集进行额外的排序，使用 EXPLAIN 可以发现 Extra 部分显示 Using filesort：
```sql
EXPLAIN SELECT name FROM user ORDER BY name DESC, age DESC;
```
![image.png](<../images/15 MySQL 进阶-SQL 语句优化/1.png>)<br />对于上述情况，我们可以在创建索引时，按照降序排序：
```sql
ALTER TABLE `user` ADD INDEX `idx_name_age` (`name` DESC, `age` DESC);
```
# 10 GROUP BY 优化
GROUP BY 通常与 HAVING 一起使用，表示先分组再根据条件过滤数据：
```sql
SELECT user_id, user_name FROM `order` GROUP BY user_id, user_name HAVING user_id > 200;
```
这种写法是先对所有记录进行分组，然后再过滤，而分组是一个耗时的操作，因此我们可以先使用 WHERE 过滤，缩小数据的范围，再进行分组：
```sql
SELECT user_id, user_name FROM `order` WHERE user_id > 200 GROUP BY user_id, user_name;
```
另外，在 MySQL 8 之前，GROUP BY 之后会隐式地加上一个 ORDER BY，来对 GROUP BY 列排序，所以对于上面的语句，假如 order 表没有建立联合索引 (user_id, user_name)，就会产生文件排序：
```sql
EXPLAIN SELECT user_id, user_name FROM `order` WHERE user_id > 200 GROUP BY user_id, user_name;
```
![image.png](<../images/15 MySQL 进阶-SQL 语句优化/2.png>)<br />因此如果我们没有排序的需求，可以加上 ORDER BY NULL：
```sql
SELECT user_id, user_name FROM `order` WHERE user_id > 200 GROUP BY user_id ORDER BY NULL;
```
# 11 将大事务拆分为小事务
可以避免事务锁长时间占用系统资源，提高系统的并发性。
