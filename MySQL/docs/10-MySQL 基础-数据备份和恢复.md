# 1 逻辑备份
逻辑备份导出的是 SQL 语句。

1. 首先加上全局锁，防止备份的时候有数据写入，并将缓存中的数据刷新到磁盘中：
```sql
FLUSH TABLES WITH READ LOCK;
```

2. 导入、导出命令：
```bash
# 导出单个表
mysqldump -u 用户名 -p 数据库名 表名 > 路径/文件名.sql 
# 导出整个数据库
mysqldump -u 用户名 -p -B 数据库名 > 路径/文件名.sql
# 导出多个数据库
mysqldump -u 用户名 -p --databases 数据库名1... > 路径/文件名.sql
# 导出所有数据库
mysqldump -u 用户名 -p --all-databases > 路径/文件名.sql
# 可以再加上 -h，表示将远程主机的数据导出到当前主机上，在此之前需要先在远程主机上配置好远程登陆的用户
mysqldump -h 远程主机地址 -u 远程主机用户名 -p 数据库名 > 路径/文件名.sql

# 导入整个数据库
mysql -u 用户名 -p < 路径/文件名.sql
# 导入表
mysql -u 用户名 -p 数据库名 < 路径/文件名.sql
# 可以再加上 -h，表示将当前主机导出的文件导入到远程主机上，在此之前需要先在远程主机上配置好远程登陆的用户
mysql -h 远程主机地址 -u 远程主机用户名 -p < 路径/文件名.sql
```

3. 然后再将锁释放：
```sql
UNLOCK TABLES;
```
另外，如果在执行导入、导出命令时报错：The MySQL server is running with the --secure-file-priv option so it cannot execute this statement，说明导入、导出被禁止或者限制了导入、导出的路径，需要在配置文件中配置：
```
[mysqld]
secure_file_priv='' # 不限制导入、导出的路径
```
# 2 物理备份
物理备份导出的是表中数据。
```sql
# 导出表中数据
select * from 表名 into outfile "路径/文件名.txt";

# 导入一张表的数据
load data infile "路径/文件名.txt" into table 库名.表名;
```
至于数据库、表结构就单独在另一台主机上执行建库、建表语句吧。
