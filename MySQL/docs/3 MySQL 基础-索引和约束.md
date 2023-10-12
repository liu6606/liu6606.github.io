以下都基于 InnoDB 存储引擎。
# 1 索引的分类
根据使用的数据结构的不同，可以将索引分为四类：

1. BTREE：使用 B+ 树作为索引，又可再分为聚簇索引和二级索引两种，实际上 MySQL 的一张表就是一个聚簇索引，也就是所谓的索引即数据、数据即索引。
2. HASH：我们无法显式地创建一个 HASH 索引，它由 InnoDB 存储引擎自动创建。
3. FULLTEXT：全文索引。
4. SPATIAL：对空间数据的索引。

按字段特性分类，可分为：

1. 主键索引（PRIMARY KEY）：每一张表都必须有一个主键，主键列会默认创建一个唯一索引，并且主键列必须设置 NOT NULL 约束。
2. 普通索引（INDEX、KEY）：建立在普通字段上的索引。
3. 唯一索引（UNIQUE [INDEX|KEY]）：唯一索引列的值不能重复，但是可以有多个 NULL 值。
4. 前缀索引：
   1. 对文本字符串类型的前几个字符、或者对二进制字符串的前几个字节建立索引，而不是对整个列建立索引。
   2. 当列的数据类型为字符串类型时，有些情况下必须建立前缀索引，详见“数据类型一节”。

按索引字段的个数分类，可分为：

1. 单列索引：建在单个列上的索引。
2. 联合索引：建在多个列上的索引。
# 2 创建索引
```sql
CREATE TABLE 表名(
  索引列名...,
  索引类型 [索引名] (索引列名1, 列名2...)
); -- 索引名不写则为第一个索引列名

CREATE 索引类型 索引名 ON 表名(索引列名1, 列名2...);

ALTER TABLE 表名 ADD INDEX [索引名](索引列名1, 列名2...);
```
需要注意：对于 TINYTEXT、TEXT、MEDIUMTEXT、LONGTEXT、TINYBLOB、BLOB、MEDIUMBLOB、LONGBLOB 类型，不能直接在相应列上建立索引，需要建立前缀索引：
```sql
CREATE TABLE IF NOT EXISTS `user` (
  `name` BLOB,
	INDEX n(`name`(2)) -- 如果是字符串类型，表示只对前两个字符建立索引；如果是二进制类型，表示只对前两个字节建立索引
);
```
并且注意索引列值的长度最长为 3072 字节，因此如果 VARCHAR、VARBINARY 列可保存的最大字节长度超过 3072 字节时，也不能直接建立索引，需要建立前缀索引。
# 3 删除索引
```sql
DROP INDEX 索引名 ON 表名;
ALTER TABLE 表名 DROP INDEX 索引名;
ALTER TABLE 表名 DROP PRIMARY KEY; -- 删除主键索引
ALTER TABLE 表名 DROP INDEX 列名; -- 删除唯一索引
```
# 4 外键约束
如果外键列关联的主表的列不存在某个值，则外键列也不能添加该值，防止添加无效的数据。
## 4.1 添加外键约束
```sql
CREATE TABLE 表名(
  外键列名 数据类型,
  FOREIGN KEY (外键列名) REFERENCES 主表名(列名)
);  -- 将外键列关联到主表中的指定 主键列

ALTER TABLE 表名 ADD FOREIGN KEY (外键列名) REFERENCES 主表名(列名);
```
## 4.2 删除外键约束
```sql
ALTER TABLE 表名 DROP FOREIGN KEY 外键名;
```
## 4.3 外键级联
外键列关联到的表称为主表，含外键列的称为从表。如果主表试图删除或更新从表中存在的外键值，则最终**删除**或**更新**成功与否以及从表的行为，取决于从表外键级联的设置。
```sql
FOREIGN KEY (外键列名) REFERENCES 主表名(列名) ON DELETE/UPDATE [options];
```
options 包括：

- RESTRICT、NO ACTION：拒绝删除或更新主表。
- CASCADE：主表删除或更新一行时，从表删除或修改外键值匹配的行。
- SET NULL：主表删除或更新一行时，从表匹配的外键列值设为 NULL，外键列不能设置 NOT NULL 约束。
# 5 其他约束
比如 NOT NULL、AUTO_INCREMENT 等，可以在创建表时指定，也可以使用 ALTER TABLE 语句：
```sql
ALTER TABLE 表名 MODIFY 列名 数据类型 约束名;
```
这个既可以用来添加约束，也可以删除约束，只要不加上已有的约束名，就会把已有的约束删除。

