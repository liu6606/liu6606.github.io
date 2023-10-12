> [http://c.biancheng.net/mysql/](http://c.biancheng.net/mysql/)

# 1 操作数据库
## 1.1 创建数据库
```sql
CREATE DATABASE IF NOT EXISTS `test` -- 创建 test 数据库，如果数据库已存在则什么也不做
CHARACTER SET utf8mb4 -- 设置默认的字符集
COLLATE utf8mb4_general_ci; -- 设置默认的排序规则
```
## 1.2 查看数据库
```sql
SHOW DATABASES LIKE "%test%"; --使用正则表达式，查找名称中包含 test 的数据库
```
## 1.3 查看某个数据库的创建语句
```sql
SHOW CREATE DATABASE `test`; -- 显示 test 数据库的创建语句
```
## 1.4 选择使用某数据库
```sql
USE `test`; -- 选择使用 test 数据库，那么之后我们对表的操作都是针对该数据库
```
## 1.5 修改数据库的默认字符集和排序规则
```sql
ALTER DATABASE `test` CHARACTER utf8mb4 COLLATE utf8mb4_general_ci; -- 需要先执行 USE `test`; 切换到 test 数据库
```
## 1.6 删除数据库
```sql
DROP DATABASE IF EXISTS `test`; -- 如果 test 数据库存在，则删除
```
# 2 操作表
## 2.1 创建表
```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT, -- 指定列名、数据类型、约束
  `name` varchar(20) DEFAULT NULL, -- 默认值 NULL
  `age` int(3) DEFAULT NULL,
  PRIMARY KEY (`id`), -- 设置主键
  KEY `age` (`age`) -- 添加索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci; -- 设置存储引擎、默认的字符集和排序规则
```
另外，只要在 CREATE 后加上 TEMPORARY，就可以创建一个临时表，临时表只在当前连接中有效。
## 2.2 复制表结构
```sql
CREATE TABLE `user2` LIKE `user`; -- 创建表 user2，并且表结构与 user 表相同
```
## 2.3 复制表数据
```sql
INSERT INTO `user2` SELECT * FROM `user`; -- 将 user 表的数据复制到 user2 表中
```
## 2.4 查看表
```sql
SHOW TABLES LIKE "%user%"; -- 查找名称中包含 user 的表
```
## 2.5 查看表结构
```sql
DESC `user`; -- 查看 user 表的结构
SHOW COLUMNS FROM `user`;
```
## 2.6 查看某个表的创建语句
```sql
SHOW CREATE TABLE `user`; -- 查看创建 user 表的语句
```
## 2.7 修改表
```sql
ALTER TABLE `user` ADD COLUMN `country` VARCHAR(10); -- 添加 country 列，数据类型为 VARCHAR(10)
ALTER TABLE `user` CHANGE COLUMN `name` `nickname` VARCHAR(30); -- 将 name 列的列名修改为 nickname，数据类型改为 VARCHAR(30)
ALTER TABLE `user` ALTER COLUMN `name` SET DEFAULT "无名氏"; -- 给 name 列设置默认值 "无名氏"
ALTER TABLE `user` ALTER COLUMN `name` DROP DEFAULT; -- 删除 name 列的默认值
ALTER TABLE `user` MODIFY COLUMN `name` VARCHAR(30); -- 修改 name 列的数据类型为 VARCHAR(30)
ALTER TABLE `user` DROP COLUMN `name`; -- 删除 name 列
ALTER TABLE `user` RENAME TO `student`; -- 将 user 表的名字修改为 student
ALTER TABLE `user` CHARACTER SET utf8mb4; -- 修改默认字符集为 utf8mb4
ALTER TABLE `user` COLLATE utf8mb4_general_ci; -- 修改默认排序规则为 utf8mb4_general_ci
```
## 2.8 删除表
```sql
DROP TABLE IF EXISTS `user`; -- 如果 user 表存在，则将其删除
```
