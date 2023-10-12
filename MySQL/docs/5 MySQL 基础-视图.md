# 1 什么是视图
视图引用了一个 SELECT，是对 SELECT 语句的封装，是一个虚拟的表，它不保存任何数据，每次查询视图，都会执行它封装的 SELECT 语句。<br />视图具有以下优点：

- 简化复杂查询：将复杂查询封装成视图，我们可以供他人使用，而他人不需要理解其中复杂的逻辑。
- 提高安全性：视图可以将表的指定字段展示给用户，而不需要将所有字段一起展示给用户，通过授予视图的只读权限给用户，不允许用户访问原始表，保障了表中数据的安全性。
- 重用 SQL 语句：将 SQL 语句用视图封装后，可以方便地重用。

但视图也有缺点：

- 操作视图会比直接操作基本表要更慢。
- 存在修改限制，某些视图是不可修改的。
# 2 创建视图及基本使用
有一个 user 表：<br />![image.png](<../images/5 MySQL 基础-视图/1.png>)<br />由于用户的地址信息属于隐私，我们不希望普通用户能够访问到该信息，因此可以创建一个视图，屏蔽 address 列，并且撤销用户访问原始表的权限，只授予读视图的权限：

1. 首先创建一个视图：
```sql
CREATE VIEW my_view AS SELECT `name`, age FROM `user`; -- 只返回 user 表的 name、age 列，屏蔽了 address 列
CREATE OR REPLACE VIEW my_view AS SELECT `name`, age FROM `user`; -- 如果视图不存在，则会创建；否则用新视图替换掉旧视图
```

2. 创建一个用户（如果配置文件中有 skip-grant-tables 选项，要删掉）：
```sql
CREATE USER 'normal_user'@'localhost' IDENTIFIED BY '123'; -- 一开始用户不具有任何权限
```

3. 授予 normal_user 用户读视图的权限：
```sql
GRANT SELECT ON test.my_view TO 'normal_user'@'localhost';
```

4. 可以像访问表一样访问视图，并且由于 normal_user 用户不具有访问原始表的权限，保障了数据的安全性：
```sql
SELECT * FROM my_view;
```
对于创建视图中的 SELECT 语句存在以下限制：

- 用户除了拥有 CREATE VIEW 权限外，还需要具有操作中涉及的基础表和其他视图的相关权限。
- SELECT 语句不能引用系统或用户变量。
- SELECT 语句不能包含 FROM 子句中的子查询。
- SELECT 语句不能引用预处理语句参数。
# 3 查看视图信息
所有视图都存储在 information_schema 数据库下的 views 表中：
```sql
SELECT * FROM information_schema.views WHERE table_name LIKE "%my_view%"; -- 查找名称中包含 my_view 的视图
```
# 4 查看视图结构
```sql
DESC my_view;
```
# 5 查看某个视图的创建语句
```sql
SHOW CREATE VIEW my_view;
```
# 6 删除视图
```sql
DROP VIEW my_view;
```
# 7 替换视图
```sql
ALTER VIEW my_view AS SELECT `name` FROM `user`;
```
# 8 修改视图内容
部分视图是可更新的，对视图数据的更新实质上是在更新视图所引用的基本表的数据。
```sql
CREATE VIEW my_view AS SELECT `name`, age FROM `user`;
-- 往视图添加记录，实际是往 user 表添加一条记录，name 列值为 Lisa、age 列值为 23。另外，user 表中的其他列，如果是 NOT NULL，则需要有默认值
INSERT INTO my_view VALUES("Lisa", 23);
```
但也有部分视图是不可更新的，因为这些视图的更新不能唯一有意义地转换为相应的基本表，如果视图包含以下结构中的任何一种，它就是不可更新的：

- 聚合函数 SUM()、MIN()、MAX()、COUNT() 等。
- DISTINCT 关键字。
- GROUP BY 子句。
- HAVING 子句。
- UNION 或 UNION ALL 运算符。
- 位于选择列表中的子查询。
- FROM 子句中的不可更新视图或包含多个表。
- WHERE 子句中的子查询，引用 FROM 子句中的表。
- ALGORITHM 选项为 TEMPTABLE（使用临时表总会使视图成为不可更新的）。
