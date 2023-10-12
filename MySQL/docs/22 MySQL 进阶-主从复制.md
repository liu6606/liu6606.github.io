>  [https://www.cnblogs.com/f-ck-need-u/p/9155003.html](https://www.cnblogs.com/f-ck-need-u/p/9155003.html)
> [https://juejin.cn/post/7169834984034402334](https://juejin.cn/post/7169834984034402334)
> [https://juejin.cn/post/7170202668013453319](https://juejin.cn/post/7170202668013453319)

# 1 基本概念及原理
MySQL 的主从复制是基于 binlog 日志的，主节点（master）将 binlog 变更的数据发送给从节点（slave），然后从节点将这些数据写入到自身的数据库中，以达到和 master 数据同步的目的。<br />具体流程如下：

1. 当从节点连接上主节点后，主节点上会启动一个 dump 线程，用来监听 binlog 日志，当主节点的数据被修改，写入 binlog 日志后，便会通知从节点来拉取变更的 binlog。
2. 从节点上存在两个线程：
   1. IO 线程：用于连接 master、从 master 接收数据（变更的 binlog），并将这些数据写入到自身的中继日志（relay log） 中。
   2. SQL 线程：用于监控、重放 relay log，当自身的 relay log 发生改变后，便将其中的数据写入到自身的数据库中，从而与 master 的数据保持一致。

另外，主从复制还提供了读写分离的能力，由于 master 和 slave 的数据都是一致的，因此可以将读请求发送给 slave、写请求发送给 master，来分散压力，提高性能。
# 2 主从架构搭建
## 2.1 传统复制方式
传统复制方式是基于 binlog 的 position 进行复制。slave 需要指定 master 的某个 binlog 的 position，那么就会从该位置开始复制。

1. 主库配置文件：
```bash
[mysqld]
# 主库在集群中的唯一标识
server-id=1
# 开启 binlog 日志，文件名为 mysql-bin-log，后缀是 .000001、.000002、...
log-bin=mysql-bin-log
# 指定 binlog 日志格式为 mixed
binlog_format=mixed
```

2. 从库配置文件：
```bash
[mysqld]
# 从库在集群中的唯一标识
server-id=2
# 开启 relay log 日志，文件名为 mysql-relay-log，后缀是 .000001、.000002、...
relay_log=mysql-relay-log
# 从库开启 binlog 日志的目的是做主从切换，也就是 master 故障时可以将 slave 切换为新的 master
log-bin=mysql-bin-log
binlog_format=mixed
log-slave-updates=true # 从库重放 relay log 的同时也写入自身 binlog 中
```

3. 首先我们在主库中创建一个用于复制的用户，并授予 REPLICATION SLAVE 权限：
```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY '123';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES; -- 一定要加上这一句，不然后面会出错
```

4. 在主库中创建一个数据库 test。
5. 从库执行以下语句设置连接参数，执行完后将在数据目录下生成 master.info 文件，保存这些参数：
```sql
change master to master_host='163.18.60.21', -- master 的 ip
       master_user='repl',
       master_password='123',
       master_port=3306,
       master_log_file='mysql-bin-log.000001', -- 第一个 binlog 文件
       master_log_pos=0; -- 复制的 position
```

6. 从库启动 IO、SQL 线程：
```sql
start slave;
```

7. 查看从库线程状态：
```sql
show slave status\G;
```

8. 一定要看到 Slave_IO_Running、Slave _SQL_Running 这两项为 Yes 才算成功：

![image.png](<../images/22 MySQL 进阶-主从复制/1.png>)

9. 在从库中执行以下命令：
```sql
show databases;
select user, host from mysql.user;
```

10. 可以发现我们在主库中创建的 repl 用户、test 数据库，都同步到了从库中。
## 2.2 GTID 复制
### 2.2.1 概念
在传统复制方式中，当发生主从切换时，从节点需要执行 change master to ... 语句连接到新的主节点，并重新确定复制的 position。比如现在 master 故障，将 slave1 切换为新的主节点，那么 slave2 就需要确定从 slave1 的 binlog 的哪个位置开始复制，也就是要找到 slave2 当前复制到的位置，对应的是 slave1 的 binlog 的哪个 position，由于不同从节点同步的进度可能不同，所以这就需要通过查看 slave1 和 slave2 的 binlog 内容来确定。<br />GTID 帮我们解决了这个问题，不再需要我们手动查找 position。<br />GTID 全称为 Global Transaction ID，即全局事务标识符，它由节点的 UUID 和事务 ID 两部分组成：

1. 节点的 UUID：MySQL 在第一次启动时会生成一个 server-uuid，保存在数据目录下的 auto.cnf 文件中。
2. 事务 ID：MySQL 会为每个写事务分配一个顺序递增的 ID。

其工作方式如下：

1. master 会为每个写事务分配一个 GTID，在向 binlog 写入事务本身之前，先写入 GTID。如下所示，GTID_NEXT 的值就是为下一个事务分配的 GTID：

![image.png](<../images/22 MySQL 进阶-主从复制/2.png>)

2. slave 的 IO 线程将接收到的 master 的 binlog 日志写入 relay log 中，并且对于保存了 GTID 的记录，执行上图中的 SET 语句，也就是设置变量 GTID_NEXT 的值。
3. slave 的 SQL 线程重放 relay log，从中读取 GTID，查找自身的 binlog 中是否有相同的 GTID，如果有，说明相应的事务已经执行过了，则忽略该事务，这保证了从服务器不会重复执行同一件事务。
4. 主从切换时，其他从库会根据自身的 GTID_NEXT 变量的值，从新的主库的 binlog 中找到正确的 position，然后继续从新的主库中复制数据。

由于 GTID 复制是基于事务的，因此只对支持事务的存储引擎生效，比如 InnoDB。
### 2.2.2 配置

1. 跟传统复制方式相比，只需在主库、从库的配置文件中都增加以下内容：
```bash
[mysqld]
# 开启GTID复制
gtid_mode=on
enforce-gtid-consistency=true
```

2. 然后从库设置连接参数的语句也不同：
```sql
change master to master_host='163.18.60.21',
       master_user='repl',
       master_password='123',
       master_port=3306,
			 master_auto_position=1;
```
如果执行 show slave status\G 语句时发现 1236 错误、或者主库已有的数据没有同步到从库，那么按如下步骤解决：

1. 主库执行以下语句，找到 gtid_purged 的值：
```sql
 show global variables like '%gtid%';
```

2. 从库执行以下语句：
```sql
stop slave;
reset slave all;
reset master;
set @@global.gtid_purged='gtid_purged 的值';
change master to master_host='163.18.60.21',
       master_user='repl',
       master_password='123',
       master_port=3306,
			 master_auto_position=1;
start slave;
```
# 3 主从切换
对于 GTID 复制，选择 GTID_NEXT 变量值最大的从节点（保证它的数据是所有从节点中最完善的），将其升级为新的主节点。<br />新的主节点执行以下语句：
```sql
stop slave;
reset slave all; -- 主要是删除 master.info 文件
```
然后其他从节点执行 change master to ... 语句连接到新的主节点。
