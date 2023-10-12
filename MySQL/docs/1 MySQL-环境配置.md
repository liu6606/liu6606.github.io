#  1 Ubuntu18 安装配置 MySQL 5.7
> VirtualBox 下载：[https://www.virtualbox.org/wiki/Download_Old_Builds_6_1](https://www.virtualbox.org/wiki/Download_Old_Builds_6_1)
> Ubuntu18 下载：[https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/18.04.6/](https://mirrors.tuna.tsinghua.edu.cn/ubuntu-releases/18.04.6/)

## 1.1 VirtualBox 配置

1. 点击新建，选择虚拟机保存的路径一直下一步：

![image.pngnice柳](<../images/1 MySQL-环境配置/1.png>)

2. 点击指定虚拟机 -> 设置 -> 存储，选择 Ubuntu 的 iso 文件：

![好的](<../images/1 MySQL-环境配置/2.png>)

3. 点击工具 -> 网络：

![](<../images/1 MySQL-环境配置/3.png>)

4. 查看是否是如下配置：

![](<../images/1 MySQL-环境配置/4.png>)

5. 点击指定虚拟机 -> 设置 -> 网络：

![image.png](<../images/1 MySQL-环境配置/5.png>)

6. 默认情况下网卡 2 没有配置，必须要使用如下配置，否则主机无法 ping 通虚拟机：

![image.png](<../images/1 MySQL-环境配置/6.png>)

7. 然后启动虚拟机，安装 Ubuntu 系统。
8. 安装完成后，打开虚拟机，点击上方菜单栏设备 -> 安装增强功能：

![image.png](<../images/1 MySQL-环境配置/7.png>)

9. 开启共享粘贴板：

![image.png](<../images/1 MySQL-环境配置/8.png>)
## 1.2 安装 MySQL 5.7

1. 更新 apt 源：
```bash
sudo apt update
```

2. 安装 MySQL：
```bash
sudo apt install mysql-server-5.7
```

3. 安装 vim：
```bash
sudo apt install vim
```

4. 修改 MySQL 配置文件 /etc/mysql/my.cnf：
```bash
sudo vim /etc/mysql/my.cnf
```

5. 按 a 键进入编辑模式，配置文件写入以下内容后，按 Esc 键退出编辑，输入 :wq 保存文件并退出 vim：
```
[mysql]
default-character-set=utf8mb4

[mysqld]
skip-name-resolve
port=3306
max_connections=200
character-set-server=utf8mb4
default-storage-engine=INNODB
lower_case_table_names=1 # 数据库、表名不区分大小写
max_allowed_packet=16M # 客户端发来的数据包的最大大小
secure_file_priv='' # 不限制导入、导出的路径
bind-address=0.0.0.0 # 这是用来允许远程登陆的
```

6. 重启 MySQL：
```bash
sudo service mysql restart
```

7. 执行以下命令登录 MySQL：
```bash
sudo mysql
```

8. 修改 root 用户密码：
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123';
```

9. 输入 exit; 退出。
10. 之后再登陆 MySQL 需要执行以下命令，输入刚刚设置的密码即可：
```bash
mysql -u root -p
```
## 1.3 配置远程登陆
注意前面配置文件中一定要配置 bind-address=0.0.0.0。

1. 执行 ip addr 命令，查看虚拟机 ip，如下所示，ip 为 192.168.56.102：

![image.png](<../images/1 MySQL-环境配置/9.png>)

2. 主机和虚拟机测试是否能 ping 通对方。
3. 创建一个用户并授予权限，用于远程登录：
```sql
CREATE USER 'remote'@'%' IDENTIFIED WITH mysql_native_password BY '123';
GRANT ALL ON *.* TO 'remote'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

4. 主机执行 mysql -h 192.168.56.102 -u remote -p，连接虚拟机上的 MySQL。

Ubuntu18 中防火墙是默认关闭的，如果在其它系统中远程登陆出现错误，注意查看防火墙是否开放了 3306 端口。
# 2 Windows 安装 MySQL 5.7
> MySQL 下载：[MySQL :: Download MySQL Community Server (Archived Versions)](https://downloads.mysql.com/archives/community/)

1. 配置环境变量：系统变量的 Path 中添加 MySQL 的 bin 目录路径。
2. MySQL 安装目录下新建 my.ini 文件，新建 data 文件夹。
3. my.ini 文件内容：
```
[mysql]
default-character-set=utf8mb4

[mysqld]
port=3306
basedir=D:\Environment\MySQL\mysql-5.7.43-winx64\ 
datadir=D:\Environment\MySQL\mysql-5.7.43-winx64\data\
max_connections=20
character-set-server=utf8mb4
default-storage-engine=INNODB
lower_case_table_names=1 # 数据库、表名不区分大小写
secure_file_priv='' # 不限制导入、导出的路径
```

4. 执行以下命令，安装 MySQL 服务：
```bash
mysqld -install
```

5. 初始化数据库文件，将会在 data 目录下生成默认的数据库及其他文件，并且设置一个空密码：
```bash
mysqld --initialize-insecure --user=mysql
```

6. 启动 MySQL 服务：
```bash
net start mysql
```

7. 登录，要求输入密码时直接回车即可：
```bash
mysql -u root -p
```

8. 修改密码为 123：
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123';
```
