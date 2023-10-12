# 1 用户管理
## 1.1 创建用户
用户由用户名+主机名构成，限定用户只能从指定主机登录。<br />如果不指定主机名，则默认为 %，表示任何主机都可登录；如果不指定密码则没有密码。
```sql
CREATE USER '用户名'@'主机名' IDENTIFIED BY '密码';
```
使用该方式创建的用户，一开始不具有任何权限。
## 1.2 删除用户
```sql
DROP USER '用户名'@'主机名';
```
## 1.3 修改密码
```sql
ALTER USER '用户名'@'主机名' IDENTIFIED WITH mysql_native_password BY '密码';
```
## 1.4 查看所有用户
用户名保存在 mysql.user 表中，查询该表即可：
```sql
USE mysql;
SELECT * FROM user;
```
## 1.5 忘记密码
在 mysql 配置文件中，[mysqld] 下添加 skip-grant-tables，重启 mysql 服务，即可免密登录，再设置密码。
## 1.6 用户重命名
```sql
RENAME USER '旧用户名'@'旧主机名' TO '新用户名'@'新主机名';
```
# 2 权限管理
## 2.1 查询用户权限
```sql
SHOW GRANTS FOR '用户名'@'主机名';
```
## 2.2 授予权限
```sql
GRANT 权限列表 ON 数据库名.表名 TO '用户名'@'主机名' [WITH GRANT OPTION]; -- 数据库名、表名可以使用通配符 *
```
WITH GRANT OPTION 表示用户可以将自己具有的权限再授予其他用户。
## 2.3 撤销权限
```sql
REVOKE 权限列表 ON 数据库名.表名 FROM '用户名'@'主机名';
```
## 2.4 可使用的权限
![图片拼接.png](<../images/8 MySQL 基础-用户及权限管理/1.png>)
