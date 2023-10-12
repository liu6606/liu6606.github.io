# 1 SELECT
## 1.1 简单查询
```sql
SELECT * FROM `user`; -- 查询 name 表的所有列
```
## 1.2 别名 AS
```sql
SELECT s.name AS n FROM `student` AS s; -- 使用 AS 给表或列起别名
SELECT s.name n FROM `student` s; -- AS 可以省略
```
## 1.3 去重
```sql
SELECT DISTINCT * FROM student; -- 对于重复的记录，只保留一个
```
## 1.4 区分大小写
如果表的排序规则是不区分大小写的，我们也可以在 SELECT 语句中指定这条语句区分大小写：
```sql
SELECT * FROM `user` WHERE name=BINARY "Jack"; -- 只匹配 Jack，jack、JACK 都不匹配
SELECT * FROM `user` WHERE name="Jack" COLLATE utf8mb4_bin;
```
## 1.5 连接查询
### 1.5.1 概述
连接查询就是从两个表中查询，以某个表作为驱动表，对于驱动表中的每条记录 A，根据指定的条件，从另一个表（称为被驱动表）中找到与记录 A 匹配的记录 B，将两条记录连接在一起，返回到结果集中。<br />连接查询分为 INNER JOIN、LEFT JOIN、RIGHT JOIN 三种类型。<br />创建两个表：<br />student 表：![image.png](<../images/4 MySQL 基础-增删查改/1.png>)，grade 表：![image.png](<../images/4 MySQL 基础-增删查改/2.png>)<br />如上所示，小红缺考了，因此 grade 表中没有她的成绩；此外，老师也参加了这次考试，因此 grade 表最后一条记录的 student_id 为 NULL。<br />现在我们从这两个表中取出每个学生的姓名和他的成绩，来看不同类型的连接查询的效果。
### 1.5.2 INNER JOIN
```sql
SELECT s.name, g.score FROM student AS s INNER JOIN grade as g ON s.id=g.student_id;
SELECT s.name, g.score FROM student AS s, grade AS g WHERE s.id=g.student_id; -- 这种写法跟 INNER JOIN 是等价的
```
![image.png](<../images/4 MySQL 基础-增删查改/3.png>)

- INNER JOIN 中两个表都可作为驱动表，假设这里以 student 表作为驱动表，对于 student 表中的每条记录 A，根据 ON 指定的条件（s.id=g.student_id），从 grade 表中找到 student_id 与记录 A 的 id 相等的记录 B，将两条记录连接在一起，返回到结果集中。
- 对于 INNER JOIN，如果驱动表中的记录 A 在被驱动表中没有匹配的记录，或者驱动表中没有记录与被驱动表中的记录 B 匹配，那么记录 A、B 都不会添加到结果集中。因此可以看到上面 student 表中的小红、以及 grade 表中的最后一条记录都没有出现在结果中。

对于 INNER JOIN，MySQL 会自动选择合适的表作为驱动表，我们也可以手动指定驱动表（一般不需要我们这么做），方法是使用 STRAIGHT_JOIN 代替 INNER JOIN，它会使用左侧的表作为驱动表。
### 1.5.3 LEFT JOIN
```sql
SELECT s.name, g.score FROM student AS s LEFT JOIN grade as g ON s.id=g.student_id;
```
![image.png](<../images/4 MySQL 基础-增删查改/4.png>)<br />与 INNER JOIN 区别：

- LEFT JOIN 中，左边的表作为驱动表，这里就是 student 表。
- 对于 LEFT JOIN，如果驱动表中的记录 A 在被驱动表中没有匹配的记录，那么记录 A 仍然会添加到结果集中，只不过属于被驱动表的列的列值为 NULL；如果驱动表中没有记录与被驱动表中的记录 B 匹配，那么记录 B 仍然不会添加到结果集中。因此这里 student 表中的小红添加到了结果中，但是属于 grade 表的 score 列的值为 NULL；而 grade 表中的最后一条记录仍然没有添加到结果中。
### 1.5.4 RIGHT JOIN
```sql
SELECT s.name, g.score FROM student AS s RIGHT JOIN grade as g ON s.id=g.student_id;
```
![image.png](<../images/4 MySQL 基础-增删查改/5.png>)<br />与 LEFT JOIN 区别：RIGHT JOIN 中，右边的表作为驱动表，这里就是 grade 表。<br />因此这里 score 表中的最后一条记录添加到了结果中，但是属于 student 表的 name 列的值为 NULL。
### 1.5.5 关于连接查询中的条件
连接查询中通过 ON 指定条件，条件可以分为两种：

- 涉及两表的条件：前面的例子中 s.id=g.student_id 就是涉及两表的条件，它用来匹配驱动表和被驱动表的记录。
- 涉及单表的条件：比如，s.id > 1、g.id > 1，这种条件用来在单表中过滤记录，不用作两表间记录的匹配。

比如，以下语句同时包含涉及单表、两表的条件：
```sql
SELECT s.name, g.score FROM student AS s LEFT JOIN grade as g ON s.id=g.student_id AND s.id>1 AND g.id>1;
```
![image.png](<../images/4 MySQL 基础-增删查改/6.png>)<br />其中有三个条件：s.id>1、s.id=g.student_id、g.id>1，那么这条语句的执行过程如下：

1. 将 student 表作为驱动表，从中找出 s.id>1 的记录。
2. 从上一步查找出的记录中，对于其中的每一条记录 A，根据 s.id=g.student_id 和 g.id>1 两个条件，去被驱动表 grade 中查找满足条件的记录 B。
3. 将记录 A 和记录 B 连接在一起，返回到结果集中。

需要注意的是，对于 LEFT JOIN 和 RIGHT JOIN，即使在驱动表中通过涉及单表的条件过滤掉了一些记录，也会将所有的记录添加到结果集中，只是属于被驱动表的列的列值为 NULL。因此上面 student 表中的小明也添加到了结果中，即使它被条件 s.id>1 过滤掉了，而属于 grade 表的 score 列的值为 NULL。
## 1.6 联合查询
使用 UNION 关键字，对于多个查询（SELECT）的结果，当它们的列数、列类型相同时，可以将这些结果合并到同一个结果集中，结果集中的列名使用第一个 SELECT 的列名。<br />创建两个表：<br />student 表：![image.png](<../images/4 MySQL 基础-增删查改/7.png>)，teacher 表：![image.png](<../images/4 MySQL 基础-增删查改/8.png>)
```sql
SELECT * FROM `student` UNION SELECT * FROM teacher; -- 删除重复的行，不光是两表之间重复的，也包括单表内重复的
SELECT * FROM `student` UNION ALL SELECT * FROM teacher; -- 保留重复的行
```
UNION 的结果：![image.png](<../images/4 MySQL 基础-增删查改/9.png>)，UNION ALL 的结果：![image.png](<../images/4 MySQL 基础-增删查改/10.png>)
## 1.7 子查询
子查询指的是将一个查询语句嵌套在另一个查询语句中，可以在 SELECT、UPDATE 和 DELETE 语句中使用，而且可以进行多层嵌套。
### 1.7.1 子查询的结果是多行单列、多行多列的
子查询的结果可以作为一个虚拟表，参与查询：
```sql
-- 查询员工入职日期是 2011-11-11 日之后的员工信息和部门信息
SELECT * FROM dept t1, (SELECT * FROM emp WHERE emp.`join_date` > '2011-11-11') t2 WHERE t1.id = t2.dept_id;
```
注意，此时需要给子查询返回的虚拟表起别名。
### 1.7.2 子查询的结果是 0 行单列、单行单列的
除了作为一个虚拟表外，此时子查询的结果还可以看作一个数，参与运算：

- 对于 0 行单列的情况，是 NULL：
```sql
SELECT (SELECT * FROM `user`) IS NULL; -- 如果 user 表只有一列，并且没有数据，则该语句的结果为 1（即 true）
```

- 对于单行单列的情况，就是表中唯一的值：
```sql
SELECT * FROM emp WHERE emp.salary < (SELECT AVG(salary) FROM emp); -- 查询员工工资小于平均工资的人
```
## 1.8 子句
各子句要按下面的顺序写。
### 1.8.1 WHERE
指定一组条件来筛选记录，SELECT、UPDATE、DELETE 中都可以用。
```sql
SELECT * FROM `user` WHERE id > 1 AND age < 20;
```
可以使用以下运算符：
```sql
> 、< 、<= 、>= 、= 、<>、!=、BETWEEN...AND、IN(集合) 、IS NULL、and 或 &&、or 或 ||、 not 或 !、LIKE、REGEXP
```
### 1.8.2 GROUP BY
对 SELECT 的结果按照指定列进行分组，通常与聚合函数一起使用。<br />有一个 user 表：![image.png](<../images/4 MySQL 基础-增删查改/11.png>)
```sql
SELECT `name`, COUNT(*) AS cnt FROM `user` GROUP BY `name`; -- 将 name 列值相同的行进行合并，并统计每一组的记录个数
```
![image.png](<../images/4 MySQL 基础-增删查改/12.png>)<br />另外，GROUP BY 之后加上WITH ROLLUP可以实现在分组统计数据基础上再进行相同的统计：
```sql
SELECT `name`, COUNT(*) AS cnt FROM `user` GROUP BY `name` WITH ROLLUP; -- 会在结尾添加一行，cnt 列的值为前面的值之和，即对整个表再执行一次 count(*)                    
SELECT IFNULL(`name`, "总数") as name, COUNT(*) AS cnt FROM `user` GROUP BY `name` WITH ROLLUP; -- 上面的结果，最后一行的 name 为 NULL，可以使用 IFNULL 函数优化一下
```
![image.png](<../images/4 MySQL 基础-增删查改/13.png>)
### 1.8.3 HAVING
跟 WHERE 一样指定一组条件，用于在 GROUP BY 之后筛选记录。
```sql
SELECT `name`, COUNT(*) AS cnt FROM `user` GROUP BY `name` HAVING cnt > 1; -- 按 name 列值分组，并且只保留组内记录个数 > 1 的记录
```
###  1.8.4 ORDER BY
用于对结果进行排序，SELECT、UPDATE、DELETE 中都能用。
```sql
SELECT * FROM `user` ORDER BY name ASC, age DESC; -- 先按照 name 列值升序排序，如果 name 列值相等，再按照 age 列值降序排序
```
### 1.8.5 LIMIT
限制读取、更新、删除的记录数，也就是说 SELECT、UPDATE、DELETE 中都能用。
```sql
SELECT * FROM `user` LIMIT 10, 20; -- 记录从 0 开始编号，从 10 号记录开始（包括），查询至多 20 条记录
```
# 2 INSERT
## 2.1 添加多行记录
```sql
INSERT INTO `user`(id, `name`, age) 
	VALUES (1, "Leo", 18), (2, "Jack", 19), (3, "Alice", 18);
```
用单条 INSERT 语句一次插入多条记录，要比使用多条 INSERT 语句更快。
## 2.2 添加单行记录
```sql
INSERT INTO `user` SET id=1, `name`="Leo", age=18; -- 未设置的列，如果有默认值，则列值为默认值
```
## 2.3 INSERT...SELECT
将 SELECT 查询的结果插入到指定表中，SELECT 的结果中列数、列类型需要与要插入的表一致。
```sql
INSERt INTO `user` SELECT * FROM student; -- 将 student 表的数据复制到 user 表中
```
# 3 DELETE
## 3.1 删除指定记录
```sql
DELETE FROM `user` WHERE id > 1 -- 删除 id>1 的记录
	ORDER BY age DESC -- 按照什么顺序删除
 	LIMIT 1; -- 删除几条数据
```
## 3.2 删除所有记录
```sql
DELETE FROM `user`;
TRUNCATE TABLE `user`; --效率更高
```
## 3.3 多表删除
根据指定的条件，从一个表或多个表中删除行。<br />创建两个表：<br />t1：![image.png](<../images/4 MySQL 基础-增删查改/14.png>)，t2：![image.png](<../images/4 MySQL 基础-增删查改/15.png>)
```sql
DELETE t1 FROM t1, t2 WHERE t1.id=t2.id; -- 只从 t1 表中删除 id=1 的行
DELETE t1, t2 FROM t1, t2 WHERE t1.id=t2.id; -- 分别从 t1、t2 表中删除 id=1 的行
```
# 4 UPDATE
```sql
UPDATE `user` SET `name`="Messi"
	WHERE `name`="Leo" -- 指定修改哪些记录，如果不指定，则修改所有记录
	ORDER BY age -- 按什么顺序修改记录
	LIMIT 2; -- 修改多少条记录
```
# 5 预处理语句
预处理语句用来防止 SQL 注入，它将 SQL 语句中依赖于用户输入的部分用占位符 ? 代替，先将 SQL 语句编译，然后再替换占位符。
```sql
SET @name = 'Leo';
SET @age = 19;
PREPARE stmt FROM 'SELECT * FROM user WHERE name = ? AND age = ?'; -- 定义预处理语句
EXECUTE stmt USING @name, @age; -- 用变量 @name、@age 的值替换占位符，执行预处理语句
```
删除预处理语句：
```sql
DROP prepare stmt; -- 删除后就不能再执行了
```
# 6 LIKE 和正则表达式
## 6.1 LIKE 模糊匹配
使用占位符：“_”表示单个字符，“%”表示任意个字符（包括 0 个）。
```sql
SELECT `name` FROM `user` WHERE `name` LIKE '%eo'; -- 查找 name 列值以 eo 结尾的记录
SELECT `name` FROM `user` WHERE `name` NOT LIKE '%eo'; -- 查找 name 列值不以 eo 结尾的记录
```
## 6.2 REGEXP 正则表达式
### 6.2.1 基本使用
```sql
SELECT `name` FROM `user` WHERE `name` REGEXP 'jack'; -- name 列值中包含 jack 时匹配该记录，不区分大小写
SELECT `name` FROM `user` WHERE `name` BINARY REGEXP 'Jack'; -- 区分大小写
SELECT `name` FROM `user` WHERE `name` REGEXP '.ack'; -- . 匹配单个字符
SELECT `name` FROM `user` WHERE `name` REGEXP 'jack|mark|andrew'; -- or
SELECT `name` FROM `user` WHERE `name` REGEXP '[abc] jack'; -- [] 表示匹配 [] 内字符之一，等价于 a jack|b jack|c jack
SELECT `name` FROM `user` WHERE `name` REGEXP '[^abc] jack'; -- ^ 表示匹配 [] 内字符之外的字符
SELECT `name` FROM `user` WHERE `name` REGEXP '[a-c] jack'; -- - 表示范围匹配，这里表示 a、b、c 都匹配
SELECT `name` FROM `user` WHERE `name` REGEXP '\\.'; -- 对于 .、|、[]、-、^ 等特殊字符需要使用 \\ 转义，\ 用 \\\ 表示
```
与 LIKE 区别在于：REGEXP 当列值中存在指定值时匹配，而 LIKE 匹配整个列值，如 Leo Messi 匹配 REGEXP 'Leo'，而不匹配 LIKE 'Leo'。<br />还有一些特殊的字符，可以分为以下三类。
### 6.2.2 字符类
一组预定义的字符集。

- [[:alnum:]]：表示任意单个字母或数字，同 [a-zA-Z0-9]。
- [[:alpha:]]：表示任意单个字母，同 [a-zA-Z]。
- [[:digit:]]：表示任意单个数字。
- [[:upper:]]：表示任意大写字母，但是首先得能区分大小写，否则等同于 [[:alpha:]]。
- [[:lower:]]：表示任意小写字母，但是首先得能区分大小写，否则等同于 [[:alpha:]]。
- [[:blank:]]：表示空格字符。
- [[:space:]]：表示任意空白字符。
- [[:xdigit:]]：表示任意 16 进制数字，同 [a-fA-F0-9]。
- [[:print:]]：表示任何可打印字符。
- [[:graph:]]：表示任何可打印字符，除了空格。
- [[:cntrl:]]：表示 ASCII 控制字符，即 ASCII 码在 0~31 和 127 的字符。
- [[:punct:]]：表示不在 [[:alnum:]] 和 [[:cntrl:]] 中的字符。
```sql
SELECT `name` FROM `user` WHERE `name` REGEXP BINARY '[[:upper:]]eo'; -- 一个大写字母+eo
```
### 6.2.3 重复元字符
表示匹配多少个指定字符。

- *：0 个及以上，同 {0,}。
- +：1 个及以上，同 {1,}。
- ?：0 或 1 个，同 {0,1}。
- {n}：n 个。
- {n,}：n 个及以上。
- {n,m}：n~m 个，m <= 255。
```sql
SELECT `name` FROM `fruits` WHERE `name` REGEXP 'apples?'; -- s? 表示 0 或 1 个 s，即 apple 和 apples 都匹配
SELECT id FROM `user` WHERE id REGEXP '[[:digit:]]{3}'; -- 匹配连续的三个数字，如 356
```
### 6.2.4 定位符
表示匹配的位置。

- ^：整个列值的开始位置。
- $：整个列值的结尾位置。
- [[:<:]]：匹配单词的开始位置（指英语单词，只要非字母、数字，都视作分隔符）。
- [[:>:]]：匹配单词的结尾位置。
```sql
SELECT `name` FROM `user` WHERE `name` REGEXP '^Leo'; -- 当列值以 Leo 开头时匹配
SELECT `name` FROM `user` WHERE `name` REGEXP '[[:<:]]Leo'; -- 当单词以 Leo 开头时匹配，比如 Messi Leo 可以匹配
```
# 7 内置函数
## 7.1 聚合函数
```sql
1. count(列名)：统计指定列非NULL的个数。可以写count(*)
2. max(列名)：指定列最大值，忽略NULL。如果表为空，返回NULL
3. min(列名)：指定列最小值，忽略NULL。如果表为空，返回NULL
4. sum(列名)：指定列的值的和，忽略NULL。如果表为空，返回NULL
5. avg(列名)：指定列的值平均值，忽略NULL。如果表为空，返回NULL
```
## 7.2 数值型函数
```sql
1.ABS(x)：返回x的绝对值
2.BIN(x)：返回x的二进制
3.CEILING(x)：返回大于x的最小整数值
4.EXP(x)：返回值e（自然对数的底）的x次方
5.FLOOR(x)：返回小于x的最大整数值
6.GREATEST(x1,x2,...,xn)：返回集合中最大的值
7.LEAST(x1,x2,...,xn)：返回集合中最小的值
8.LN(x)：返回x的自然对数
9.LOG(x,y)：返回x的以y为底的对数
10.MOD(x,y)：返回x/y的模（余数）
11.PI()：返回pi的值（圆周率）
12.RAND()：返回0到1内的随机值,可以通过提供一个参数(种子)
13.ROUND(x,y)：返回参数x的四舍五入的有y位小数的值
14.TRUNCATE(x,y)：返回数字x截短为y位小数的结果
```
## 7.3 字符串函数
```sql
1.LENGTH(s)：计算字符串长度函数，返回字符串的字节长度
2.CONCAT(s1,s2...,sn)：合并字符串函数，返回结果为连接参数产生的字符串
3.INSERT(str,x,y,newstr)：将字符串str从第x位置开始，y个字符长的子串替换为字符串newstr，返回结果
4.LOWER(str)：将字符串中的字母转换为小写
5.UPPER(str)：将字符串中的字母转换为大写
6.LEFT(str,x)：返回字符串str中最左边的x个字符
7.RIGHT(str,x)：返回字符串str中最右边的x个字符
8.TRIM(str)：删除字符串左右两侧的空格
9.REPLACE(string,str,newstr)：将string中的str子串替换为newstr, 返回替换后的新字符串
10.SUBSTRING(string,start,length)：截取字符串,返回从指定位置(1开始)开始的指定长度的字符串
11.REVERSE(str)：返回颠倒字符串str的结果
```
## 7.4 时间函数
```sql
1.CURDATE 和 CURRENT_DATE   两个函数作用相同，返回当前系统的日期值
2.CURTIME 和 CURRENT_TIME   两个函数作用相同，返回当前系统的时间值
3.NOW 和 SYSDATE   两个函数作用相同，返回当前系统的日期和时间值
4.UNIX_TIMESTAMP   获取UNIX时间戳函数，返回一个以 UNIX 时间戳为基础的无符号整数
5.FROMUNIXTIME   将 UNIX 时间戳转换为时间格式，与UNIXTIMESTAMP互为反函数
6.MONTH   获取指定日期中的月份
7.MONTHNAME   获取指定日期中的月份英文名称
8.DAYNAME   获取指定曰期对应的星期几的英文名称
9.DAYOFWEEK   获取指定日期对应的一周的索引位置值
10.WEEK   获取指定日期是一年中的第几周，返回值的范围是否为 0〜52 或 1〜53
11.DAYOFYEAR   获取指定曰期是一年中的第几天，返回值范围是1~366
12.DAYOFMONTH   获取指定日期是一个月中是第几天，返回值范围是1~31
13.YEAR   获取年份，返回值范围是 1970〜2069
14.TIMETOSEC   将时间参数转换为秒数
15.SECTOTIME   将秒数转换为时间，与TIMETOSEC 互为反函数
16.DATE_ADD 和 ADDDATE   两个函数功能相同，都是向日期添加指定的时间间隔
17.DATE_SUB 和 SUBDATE   两个函数功能相同，都是向日期减去指定的时间间隔
18.ADDTIME   时间加法运算，在原始时间上添加指定的时间
19.SUBTIME   时间减法运算，在原始时间上减去指定的时间
20.DATEDIFF   获取两个日期之间间隔，返回参数 1 减去参数 2 的值
21.DATE_FORMAT   格式化指定的日期，根据参数返回指定格式的值
22.WEEKDAY   获取指定日期在一周内的对应的工作日索引
```
## 7.5 加密函数
```sql
1.ENCRYPT(str,salt)   使用UNIXcrypt()函数，用关键词salt(一个可以惟一确定口令的字符串，就像钥匙一样)加密字符串str
2.ENCODE(str,key)   使用key作为密钥加密字符串str，调用ENCODE()的结果是一个二进制字符串，它以BLOB类型存储
3.MD5()   计算字符串str的MD5校验和
4.PASSWORD(str)   返回字符串str的加密版本，这个加密过程是不可逆转的，和UNIX密码加密过程使用不同的算法。
5.SHA()   计算字符串str的安全散列算法(SHA)校验和
```
## 7.6 流程控制函数
```sql
1.IF(test,t,f)   如果test是真，返回t；否则返回f
2.IFNULL(arg1,arg2)   如果arg1不是空，返回arg1，否则返回arg2
3.NULLIF(arg1,arg2)   如果arg1=arg2返回NULL；否则返回arg1
4.CASE WHEN[test1] THEN [result1]...ELSE [default] END   如果testN是真，则返回resultN，否则返回default
5.CASE [test] WHEN[val1] THEN [result1]...ELSE [default]END   如果testN和valN相等，则返回resultN，否则返回default
```
