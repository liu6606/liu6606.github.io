# 1 查看各 SQL 语句执行频率
执行如下语句：
```sql
SHOW GLOBAL STATUS LIKE 'com_______'; -- 自 MySQL 上次启动至今的统计结果
SHOW SESSION STATUS LIKE 'com_______'; -- 当前会话的统计结果
```
![](<../images/16 MySQL 进阶-SQL 性能分析工具/1.png>)<br />该方法得到的是 MySQL 中所有数据库的 SQL 语句执行频率。
# 2 慢查询日志
慢查询日志记录了所有执行时间超过指定参数 long_query_time（默认为 10s）的所有 SQL 语句。<br />默认情况下，慢查询日志是关闭的，在 MySQL 的配置文件中开启慢查询日志：
```sql
slow_query_log = 1 -- 开启慢查询日志
long_query_time = 2 -- 设置慢日志的时间为 2s，SQL语句执行时间超过 2s，就会记录到慢查询日志中，默认为 10s
```
也可以设置变量来开启：
```sql
SET GLOBAL slow_query_log = on;
SET GLOBAL long_query_time = 2;
```
在配置好后，我们模拟一个超过2s的查询语句，去查询一个数据量为1000w的记录条数，实际操作的时间为13.350650s，而在慢查询日志中记录的数据为：<br />![](<../images/16 MySQL 进阶-SQL 性能分析工具/2.png>)<br />默认情况下，慢查询日志保存在 data 目录下，文件名为 **主机名-slow.log**。
# 3 查看 SQL 语句各阶段的耗时
通过 Profiles 可以获取 SQL 语句在它执行的各个阶段的耗时。<br />Profiles 是默认开启的，可以通过设置 profiling 变量来控制是否开启 Profiles：
```sql
SET profiling = 1; -- 开启 Profiles
```

1. 执行如下命令查看 SQL 语句的耗时：
```sql
SHOW PROFILES;
```
![](<../images/16 MySQL 进阶-SQL 性能分析工具/3.png>)

2. 根据上面的得到的 Query_ID，来获取该 SQL 语句在其执行的各个阶段的耗时：
```sql
SHOW PROFILE FOR QUERY 2;
```
![](<../images/16 MySQL 进阶-SQL 性能分析工具/4.png>)
# 4 EXPLAIN 执行计划
一条查询语句在经过 MySQL 查询优化器的各种基于成本和规则的优化会后生成一个所谓的执行计划 ，这个执行计划展示了接下来具体执行查询的方式，使用 EXPLAIN 可以查看执行计划。
## 4.1 type
表示这条查询的访问方法是什么，从最优到最差依次为：**system> const > eq_ref > ref > >ref_or_null > index_merge > range > index > ALL**。<br />一般来说，**要保证查询达到 range 级别，最好达到 ref**。
### 4.1.1 system
当表中只有一条记录并且该表使用的存储引擎的统计数据是精确的，比如 MyISAM、Memory，那么对该表的访问方法就是 system。  
```sql
CREATE TABLE t(i int) ENGINE=MyISAM; -- 存储引擎为 MyISAM

INSERT INTO t VALUES(1); -- 表中只有一条记录

EXPLAIN	SELECT i FROM t;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/5.png>)
### 4.1.2 const
当我们根据主键或者唯一二级索引列与常数进行等值匹配时，对单表的访问方法就是 const。
```sql
CREATE TABLE t(id int PRIMARY KEY);

INSERT INTO t VALUES(1);

EXPLAIN	SELECT id FROM t WHERE id=1; -- 根据主键与常数进行等值匹配
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/6.png>)
### 4.1.3 eq_ref
在连接查询时，如果被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（如果该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较），则对该被驱动表的访问方法就是 eq_ref。
```sql
CREATE TABLE t1(id int PRIMARY KEY);
CREATE TABLE t2(id int PRIMARY KEY);

INSERT INTO t1 VALUES(1);
INSERT INTO t2 VALUES(1);

EXPLAIN	SELECT * FROM t1 LEFT JOIN t2 ON t1.id=t2.id; -- 被驱动表 t2 通过主键等值匹配
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/7.png>)
### 4.1.4 ref
当通过普通的二级索引列与常量进行等值匹配时来查询某个表，那么对该表的访问方法就**可能**是 ref。
```sql
CREATE TABLE t(i int, INDEX(i));

INSERT INTO t VALUES(1);

EXPLAIN	SELECT * FROM t WHERE i=1;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/8.png>)
### 4.1.5 ref_or_null
当对普通二级索引进行等值匹配查询，该索引列的值也可以是 NULL 值时，那么对该表的访问方法就**可能**是 ref_or_null ，比如说：
```sql
CREATE TABLE t(i int, INDEX(i));

INSERT INTO t VALUES(1);

EXPLAIN	SELECT * FROM t WHERE i=1 OR i IS NULL; -- 使用 IS NULL
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/9.png>)
### 4.1.6 index_merge
使用索引合并的方式来对某个表执行查询的，那么对该表的访问方法就是 index_merge。
```sql
CREATE TABLE t(i int, j int, INDEX(i), INDEX(j));

INSERT INTO t VALUES (1, 3);
INSERT INTO t VALUES (1, 4);
INSERT INTO t VALUES (3, 2);
INSERT INTO t VALUES (4, 2);

EXPLAIN	SELECT * FROM t WHERE i=1 AND j=2;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/10.png>)<br />如上所示，可以使用索引合并，从 i 的索引中取出满足 i=1 的记录，从 j 的索引中取出满足 j=2 的记录，然后再对结果求交集。
### 4.1.7 range
如果使用索引获取某些范围区间的记录，那么就**可能**使用到 range 访问方法。
```sql
CREATE TABLE t(i int, INDEX(i));

INSERT INTO t VALUES (1);
INSERT INTO t VALUES (2);

EXPLAIN	SELECT * FROM t WHERE i>1;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/11.png>)
### 4.1.8 index
当我们可以使用索引覆盖，但需要扫描全部的二级索引记录时，该表的访问方法就是 index。
```sql
CREATE TABLE t(i int, j int, INDEX(i, j));

INSERT INTO t VALUES (1, 2);

EXPLAIN	SELECT * FROM t WHERE j=2;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/12.png>)<br />如上所示，我们需要查询的 i、j 列都位于二级索引中，因此可以使用索引覆盖，但是由于搜索条件不满足最左前缀原则，所以必须扫描二级索引中的全部记录。<br />注意：二级索引的记录只包含索引列和主键列的值，而聚簇索引中包含用户定义的全部列以及一些隐藏列，所以扫描二级索引的代价比直接全表扫描，也就是比扫描聚簇索引的代价更低一些。  
### 4.1.9 ALL
全表扫描，也就是需要扫描聚簇索引中的全部记录，这时该表的访问方法就是 ALL。
```sql
CREATE TABLE t(id int PRIMARY KEY, i int);

INSERT INTO t VALUES (1, 2);

EXPLAIN	SELECT * FROM t;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/13.png>)
## 4.2 possible_keys & key & key_len
 possible_keys 表示可能用到的索引有哪些，key 表示经过查询优化器计算使用不同索引的成本后，实际用到的索引有哪些：
```sql
CREATE TABLE t(name VARCHAR(100), INDEX(name));

INSERT INTO t VALUES ("Leo");

EXPLAIN	SELECT * FROM t WHERE name="Leo";
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/14.png>)<br />key_len 列表示实际用到的索引列的最大长度，之所以是最大长度，是因为存在变长字符串类型，其占用的空间大小不是固定的，如上所示，name 列为 VARCHAR(100) 类型，编码为 utf8mb4，对于变长字符串来说，会有 2 个字节的额外空间来存储该变长列的实际长度，另外，列值允许 NULL 还需要多占用一个字节，因此最终 key_len 为 100*4+2+1=403。
## 4.3 ref
当使用索引列等值匹配的条件去执行查询时，也就是在访问方法是 const 、 eq_ref 、 ref 、 ref_or_null 、 unique_subquery 、 index_subquery 其中之一时， ref 列展示的就是与索引列作等值匹配的东东是个啥，比如只是一个常数或者是某个列。
```sql
CREATE TABLE t(id int PRIMARY KEY);

INSERT INTO t VALUES(1);

EXPLAIN	SELECT * FROM t WHERE id=1; -- 使用常数 1 跟索引列作等值匹配
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/15.png>)<br />如上所示，显示 const，表示用一个常数跟索引列进行等值匹配。
```sql
CREATE TABLE t1(id int PRIMARY KEY);
CREATE TABLE t2(id int PRIMARY KEY);

EXPLAIN	SELECT * FROM t1 JOIN t2 ON t1.id=t2.id; -- 被驱动表 t2 使用 t1.id 列跟它的 t2.id 列作等值匹配
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/16.png>)<br />如上所示，显示一个列名，表示使用某个列跟索引列进行等值匹配。
## 4.4 Extra
顾名思义，Extra 列是用来说明一些额外信息的，我们可以通过这些额外信息来更准确的理解 MySQL 到底将如何执行给定的查询语句。
### 4.4.1 No tables used
当查询语句没有 FROM 子句时将会提示该额外信息。  
```sql
EXPLAIN SELECT 1; -- 没有 FROM 子句
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/17.png>)
### 4.4.2 Impossible WHERE
查询语句的 WHERE 子句永远为 FALSE 时将会提示该额外信息。
```sql
CREATE TABLE t(id int PRIMARY KEY);

EXPLAIN SELECT * FROM t WHERE 1!=1; -- 1!=1 永远为 false
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/18.png>)
### 4.4.3 No matching min/max row
当查询列表处有 MIN 或者 MAX 聚集函数，但是并没有符合 WHERE 子句中的搜索条件的记录时，将会提示该额外信息。 
```sql
CREATE TABLE t(id int PRIMARY KEY);

INSERT INTO t VALUES(1);

EXPLAIN SELECT min(id) FROM t WHERE id=2; -- 并没有 id=2 的行
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/19.png>)
### 4.4.4 Using index
当我们的查询列表以及搜索条件中只包含属于某个索引的列，也就是在可以使用索引覆盖的情况下，在 Extra 列将会提示该额外信息。
```sql
CREATE TABLE t(i int, KEY(i));

EXPLAIN SELECT i FROM t WHERE i=1;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/20.png>)
### 4.4.5 Using index condition
如果在查询语句的执行过程中将要使用”索引条件下推“这个特性，在 Extra 列中将会显示 Using index condition。下面两种情况都会使用索引下推：
```sql
CREATE TABLE t(id int, i varchar(10), INDEX(i));

EXPLAIN SELECT * FROM t WHERE i>"Leo" AND i LIKE "%messi";
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/21.png>)
```sql
CREATE TABLE t(id int, i int, j int, INDEX(i, j));

EXPLAIN SELECT * FROM t WHERE i>1 AND j>2;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/22.png>)
### 4.4.6 Using where
当我们使用全表扫描来执行对某个表的查询，并且该语句的 WHERE 子句中有针对该表的搜索条件时，在 Extra 列中会提示该额外信息。
```sql
CREATE TABLE t(i int); -- i 列没有建立索引

EXPLAIN SELECT * FROM t WHERE i=1; -- 那么只能进行全表扫描了
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/23.png>)<br />当使用索引访问来执行对某个表的查询，并且该语句的 WHERE 子句中有除了该索引包含的列之外的其他搜索条件时，在 Extra 列中也会提示该额外信息。  
```sql
CREATE TABLE t(id int PRIMARY KEY, i int);

EXPLAIN SELECT * FROM t WHERE id>1 AND i>2; -- 使用主键 id>1，不需要全表扫描，但是 i 列没有索引
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/24.png>)
### 4.4.7 Using join buffer (Block Nested Loop)
在连接查询执行过程中，当被驱动表不能有效的利用索引加快访问速度， MySQL 一般会为其分配一块名叫 join buffer 的内存块来加快查询速度。
```sql
CREATE TABLE t1(i int);
CREATE TABLE t2(i int);

EXPLAIN SELECT * FROM t1 JOIN t2 ON t1.i=t2.i; -- 对表 t2 的访问不能利用索引
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/25.png>)<br />可以看到，Extra 中除了 Using join buffer (Block Nested Loop) 外，还有 Using where，这是因为 t1 是驱动表，t2 是被驱动表，在访问 t2 表的时候，t1.i 的值已经确定下来了，所以实际上查询 t2 表的条件就是 t2.i=某个常数，需要对 t2 进行全表扫描，所以有 Using where。
### 4.4.8 Not exists
当我们使用左（外）连接时，如果 WHERE 子句中包含要求被驱动表的某个列等于 NULL 值的搜索条件，而且那个列又是不允许存储 NULL 值的，那么在该表的执行计划的 Extra 列就会提示 Not exists 额外信息。
```sql
CREATE TABLE t1(id int PRIMARY KEY);
CREATE TABLE t2(id int PRIMARY KEY, j int NOT NULL);

EXPLAIN SELECT * FROM t1 LEFT JOIN t2 ON t1.id=t2.id WHERE t2.j IS NULL; -- 要求 t2.j 为 NULL，但是该列为 NOT NULL 列
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/26.png>)<br />在 Not exists 的情况下，意味着必定是驱动表的记录在被驱动表中找不到匹配 ON 子句条件的记录（LEFT JOIN 中这时被驱动表的列为 NULL），才会把该驱动表的记录加入到最终的结果集，所以对于某条驱动表中的记录来说，如果能在被驱动表中找到 1 条符合 ON 子句条件的记录，那么该驱动表的记录就不会被加入到最终的结果集，也就是说我们没有必要到被驱动表中找到全部符合 ON 子句条件的记录，这样可以稍微节省一点性能。  
### 4.4.9  Using intersect、Using union、Using sort_union
分别表示将使用 Intersect、Union、Sort-Union 索引合并的方式执行查询。
```sql
CREATE TABLE t(i int, j int, INDEX(i), INDEX(j));

INSERT INTO t VALUES (1, 3);
INSERT INTO t VALUES (1, 4);
INSERT INTO t VALUES (3, 2);
INSERT INTO t VALUES (4, 2);

EXPLAIN	SELECT * FROM t WHERE i=1 AND j=2;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/27.png>)<br />如上所示，表示将使用 i、j 这两个索引进行索引合并的方式执行查询。
### 4.4.10 Zero limit
当我们的 LIMIT 子句的参数为 0 时，表示压根儿不打算从表中读出任何记录，将会提示该额外信息。
```sql
CREATE TABLE t(id int PRIMARY KEY);

EXPLAIN SELECT * FROM t LIMIT 0;
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/28.png>)
### 4.4.11 Using filesort
由于索引中的记录是排好序的，如果 ORDER BY 的顺序跟索引中记录的顺序是相同的，那么就不需要额外的排序，比如：
```sql
CREATE TABLE t(i int, j int, INDEX(i, j));

EXPLAIN SELECT * FROM t ORDER BY i, j;
```
否则取出记录后，还需要按照 ORDER BY 的规则重新排序，Extra 列将显示 Using filesort。
```sql
CREATE TABLE t(i int);

EXPLAIN SELECT * FROM t ORDER BY i; -- 聚簇索引中记录并不按 i 的顺序排序
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/29.png>)<br />如果查询中需要使用 filesort 的方式进行排序的记录非常多，那么这个过程是很耗费性能的，尽量避免。
### 4.4.12 Using temporary
在许多查询的执行过程中， MySQL 可能会借助临时表来完成一些功能，比如去重、排序之类的，比如我们在执行许多包含 DISTINCT 、 GROUP BY 、 UNION 等子句的查询过程中，如果不能有效利用索引来完成查询， MySQL 很有可能寻求通过建立内部的临时表来执行查询。如果查询中使用到了内部的临时表，在执行计划的 Extra 列将会显示 Using temporary 提示。
```sql
CREATE TABLE t(id int PRIMARY KEY, i int);

EXPLAIN SELECT i FROM t GROUP BY i; -- i 列没有建立索引，会建立一个临时表
```
![image.png](<../images/16 MySQL 进阶-SQL 性能分析工具/30.png>)<br />执行计划中出现 Using temporary 并不是一个好的征兆，因为建立与维护临时表要付出很大成本的， 所以我们最好能使用索引来替代掉使用临时表。这里只需在 i 列上建立索引即可。
## 4.5 其他列

1. id： 在一个大的查询语句中每个 SELECT 关键字都对应一个唯一的 id，id 值越大，越早执行。
2. select_type：SELECT 关键字对应的那个查询的类型。
3. partitions：匹配的分区信息。
4. rows：预估的需要读取的记录条数。
5. filtered：某个表经过搜索条件过滤后，满足条件的记录条数占搜索的记录条数的百分比。
## 4.6 JSON 格式的执行计划
EXPLAIN 后加上 FORMAT=JSON 。 这样我们就可以得到一个 json 格式的执行计划，里边儿包含该计划花费的成本。
```sql
EXPLAIN format=json SELECT id FROM t;
```
