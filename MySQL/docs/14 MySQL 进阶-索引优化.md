> [https://www.chanmufeng.com/posts/storage/MySQL/MySQL%E7%B4%A2%E5%BC%95%E7%9A%84%E6%AD%A3%E7%A1%AE%E4%BD%BF%E7%94%A8%E5%A7%BF%E5%8A%BF.html](https://www.chanmufeng.com/posts/storage/MySQL/MySQL%E7%B4%A2%E5%BC%95%E7%9A%84%E6%AD%A3%E7%A1%AE%E4%BD%BF%E7%94%A8%E5%A7%BF%E5%8A%BF.html)
> [https://juejin.cn/post/7169971269549424677?searchId=20231007211747796953939187217FFDD8](https://juejin.cn/post/7169971269549424677?searchId=20231007211747796953939187217FFDD8)
> [https://www.zhihu.com/question/421944348/answer/3155425308](https://www.zhihu.com/question/421944348/answer/3155425308)

# 1 索引下推（ICP）
这个是 MySQL 自己的优化，不需要我们管。<br />首先了解 MySQL 大概的架构：<br />![](<../images/14 MySQL 进阶-索引优化/1.png>)<br />Server 层负责 SQL 语法解析、生成执行计划等，并调用存储引擎层去执行数据的存储和检索。<br />索引下推是把本应该在 Server 层进行筛选的条件，下推到存储引擎层来进行筛选判断，这样能有效减少回表。<br />ICP 是 MySQL5.6 引入的功能，默认开启，可以执行如下语句关闭 ICP：
```sql
SET optimizer_switch="index_condition_pushdown=off";
```
执行如下的 SQL 语句：
```sql
CREATE TABLE `user_innodb` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `gender` tinyint(1) DEFAULT NULL,
  `phone` varchar(11) DEFAULT NULL,
  PRIMARY KEY (`id`)，
  INDEX IDX_NAME_PHONE (name, phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM user_innodb WHERE name > "Leo" AND phone = "6606"; -- 由于范围查询，这里只有 name 会使用索引
```
在不使用 ICP 时，查询的执行过程如下：

1. 先在二级索引中查找到所有 name>"Leo" 的数据，得到相应记录的主键值。
2. 回表，在主键索引中拿到完整的用户记录。
3. 存储引擎将找到的记录交给 Server 层，Server 层过滤出满足条件的记录。

这样做的问题在于，假如有 10 万条 name>"Leo" 的记录，但是其中只有 1 条满足条件 phone="6606"，那么 InnoDB 需要将 99999 条无效的记录进行回表，再交给 Server 层过滤，代价很大。<br />当使用 ICP 时，查询的执行过程如下：

1. 先在二级索引中查找到所有 name>"Leo" 的数据，然后存储引擎直接判断记录是否满足 phone="6606"。
2. 对于满足的记录，拿到主键去回表。
3. 存储引擎将找到的记录交给 Server 层，Server 层对其它条件进行过滤。

![image.png](<../images/14 MySQL 进阶-索引优化/2.png>)<br />Extra 显示 Using index condition，就说明使用了索引下推。<br />不过需要注意，即使满足索引下推的使用条件，查询优化器也未必会使用索引下推，因为可能存在更高效的方式。
# 2 索引失效
## 2.1 索引为什么会失效
我们知道索引是一个 B+ 树，能加快查询的关键就是 B+ 树中的记录是有序的，能够使用二分法来加快查询。<br />因此，如果一个查询不能利用“B+ 树中的记录是有序的”这个特性，那么索引就会失效。比如联合索引中所说的最左前缀原则，当有一个 (a, b, c) 的联合索引，以 (a, c) 作为查询条件时，那么只有列 a 能使用索引，列 c 索引失效，就是因为 B+ 树中的记录并不是按照列 c 有序的。<br />另外，索引失效还跟查询优化器有关，在某些情况下，查询优化器可能认为走索引还没有全表扫描快，那么就不会使用索引。
## 2.2 索引失效的情况
### 2.2.1 对索引列使用数据类型转换
MySQL 中字符串可以隐式地转换为数字类型，如：
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `name`=12345;
```
![image.png](<../images/14 MySQL 进阶-索引优化/3.png>)<br />如上所示，name 列为 varchar 类型，但是在查询中隐式地转换为了数字类型，因此索引失效。<br />而如果没有发生数据类型转换，则不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name`='12345';
```
![image.png](<../images/14 MySQL 进阶-索引优化/4.png>)<br />这是因为 B+ 树索引中的记录并不按照类型转换后的值排序，比如某行记录的 name 列值为 '2345'，类型转换后为 2345，2345 < 12345，但是 '2345' > '12345'。
### 2.2.2 在索引列上使用函数
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE LOWER(`name`)='abc';
```
![image.png](<../images/14 MySQL 进阶-索引优化/5.png>)<br />而没有使用函数时，索引不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name`='abc';
```
![image.png](<../images/14 MySQL 进阶-索引优化/6.png>)<br />这是因为 B+ 树索引中的记录并不按照函数执行结果进行排序。
### 2.2.3 索引列参与运算
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_age` (`age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `age`+1=2;
```
![image.png](<../images/14 MySQL 进阶-索引优化/7.png>)<br />而索引列没有参与运算时，不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `age`=2;
```
![image.png](<../images/14 MySQL 进阶-索引优化/8.png>)<br />这是因为 B+ 树索引中的记录并不按运算后的值进行排序。
### 2.2.4 OR 前后没有同时使用索引
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 2);

EXPLAIN SELECT * FROM `user` WHERE `name`='Leo' OR age=1;
```
![image.png](<../images/14 MySQL 进阶-索引优化/9.png>)<br />如上所示，OR 之后的 age 列上并没有建立索引，所以 name 索引失效，结果为全表扫描。<br />而在 age 上建立索引之后，name、age 索引都将生效：
```sql
ALTER TABLE `user` ADD INDEX `idx_age` (`age`);
EXPLAIN SELECT * FROM `user` WHERE `name`='Leo' OR age=1;
```
![image.png](<../images/14 MySQL 进阶-索引优化/10.png>)<br />这是因为，当 age 列上没有建立索引时，由于 OR 是左右任一条件满足即可，因此必须通过全表扫描查找满足 age=1 的记录，在这个过程中就可以顺便把 name='Leo' 的记录也找出来，没必要再走索引了，因此 name 索引失效。
### 2.2.5 模糊查询中 LIKE 以通配符开头
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `name` LIKE '%Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/11.png>)<br />而不以 % 或 _ 开头时，不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name` LIKE 'Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/12.png>)<br />这是因为索引 B+ 树中，记录的排序，是从头往后开始比较的，而当 LIKE 以通配符开头时，没有办法使用 B+ 树索引进行查询。
### 2.2.6 反向查询
当执行 !=、<>、NOT LIKE、NOT IN、NOT BETWEEN 等反向查询时，索引将失效。
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `name`!='Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/13.png>)<br />而正向查询时就不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name`='Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/14.png>)<br />因为很显然，反向查询并不能通过索引的 B+ 树加快查询。
### 2.2.7 IS NULL、IS NOT NULL 判断
当索引列允许 NULL 值，并且执行 IS NULL、IS NOT NULL 判断时，将不会走索引；当索引列不允许 NULL 值时，执行 IS NULL 显然是没有结果的，执行 IS NOT NULL 也没必要走索引，直接全表扫描返回所有记录即可。<br />这是因为 B+ 树索引会根据索引列值进行排序，而 NULL 值无法进行比较和排序。
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `user` VALUES(1, NULL, 18);
INSERT INTO `user` VALUES(2, NULL, 18);
INSERT INTO `user` VALUES(3, NULL, 18);
INSERT INTO `user` VALUES(4, 'Leo', 18);
INSERT INTO `user` VALUES(5, 'Jack', 18);
INSERT INTO `user` VALUES(6, 'Mark', 18);

EXPLAIN SELECT * FROM `user` WHERE `name` IS NULL;
```
![image.png](<../images/14 MySQL 进阶-索引优化/15.png>)<br />IS NOT NULL：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name` IS NOT NULL;
```
![image.png](<../images/14 MySQL 进阶-索引优化/16.png>)
### 2.2.8 联合索引违背最左前缀原则
假设有一个联合索引 (a, b, c)，如果违背最左前缀原则，比如对 (a, c) 进行查询，那么索引 c 将失效，索引 c 失效就意味着在 B+ 树索引中只能通过列 a 的条件进行快速查询，然后再根据列 c 的条件将结果过滤。 <br />这是因为 B+ 树索引中的记录是按照最左前缀的顺序排序的，也就是说它会按照列 a 的顺序排序，而并不按照 (a, c) 的顺序排序，因此列 c 不会使用索引。
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
	`address` varchar(10) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name_age` (`name`, `age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE age=1;
```
![image.png](<../images/14 MySQL 进阶-索引优化/17.png>)<br />如上所示，age 列并没有走索引。<br />而如果满足最左前缀原则，就可以使用索引：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name`='Leo' AND age=1;
```
![image.png](<../images/14 MySQL 进阶-索引优化/18.png>)
### 2.2.9 联合索引某列范围查询
假设有一个联合索引 (a, b, c)，如果对列 a 进行范围查询，那么索引 b、c 将失效。<br />这是因为索引中的记录，只在列 a 值相同时，才按 (b, c) 有序，而对列 a 进行范围查询，意味着 a 的值是不相同的，因此无法使用列 b、c 的条件在 B+ 树索引中查询。
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
	`address` varchar(10) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name_age` (`name`, `age`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `name`>'Leo' AND age=19;
```
![image.png](<../images/14 MySQL 进阶-索引优化/19.png>)<br />如上所示，key_len 为 42，即 10*4+2，也就是说 age 索引失效了。<br />而如果 name 不是范围查询，则不会失效：
```sql
EXPLAIN SELECT * FROM `user` WHERE `name`='Leo' AND age=19;
```
![image.png](<../images/14 MySQL 进阶-索引优化/20.png>)
### 2.2.10 不同列进行比较
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
	`address` varchar(10) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`),
	KEY `idx_address` (`address`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

EXPLAIN SELECT * FROM `user` WHERE `name`=`address`;
```
![image.png](<../images/14 MySQL 进阶-索引优化/21.png>)<br />如上所示，因为列值不是常量，显然无法通过 B+ 树来查询，只能使用全表扫描。
### 2.2.11 索引选择性低
索引选择性 = 不重复的索引列值个数（基数） / 总数，索引选择性越低，意味着重复的值越多，那么此时 MySQL 可能会放弃走索引。
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) NOT NULL,
  `age` int(11) NOT NULL,
  PRIMARY KEY (`id`),
	KEY `idx_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `user` (`name`, `age`) VALUES('Leo', 16);
INSERT INTO `user` (`name`, `age`) VALUES('Jack', 17);
INSERT INTO `user` (`name`, `age`) VALUES('Alice', 18);

EXPLAIN SELECT * FROM `user` WHERE `name`='Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/22.png>)<br />如上所示，此时 name 索引生效。<br />现在增加一些 name 列值重复的数据，使索引选择性降低，此时查询这个重复的列值，索引将失效：
```sql
INSERT INTO `user` (`name`, `age`) VALUES('Leo', 17);
INSERT INTO `user` (`name`, `age`) VALUES('Leo', 18);

EXPLAIN SELECT * FROM `user` WHERE `name`='Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/23.png>)<br />原因可能是由于索引选择性太低，回表花费的时间太多，还不如全表扫描快，因为当不需要回表时，可以发现这时又会使用索引：
```sql
EXPLAIN SELECT `name` FROM `user` WHERE `name`='Leo';
```
![image.png](<../images/14 MySQL 进阶-索引优化/24.png>)
# 3 索引设计原则

1. 首先谨慎考虑以上所以失效的情况。
2. 只为用于查询条件、排序或分组的列创建索引。
3. 频繁更新的列，不要作为索引，因为可能导致页分裂。
4. 随机无序的值，不要作为索引，因为可能导致页分裂。
5. 对过长的字段，建立前缀索引，而不是对整个列建立索引。
6. 使用自增 ID 作为主键，而不是 UUID，因为 UUID 是无序的，可能导致页分裂；另外，使用自增 ID 的话，新插入的行要插入的位置是确定的，是在原有的最大数据行的下一行（这个地址在页中单独保存了），因此定位和寻址很快。
7. 一个表建立的索引不宜过多，阿里巴巴的开发者手册中规定，单表的索引数量应该尽量控制在 5 个以内，并且单个索引中的字段数不超过 5 个。因为 INSERT、UPDATE、DELETE 操作需要更新索引，索引数量太多，也会消耗很多性能。
