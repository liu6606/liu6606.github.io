存储过程和存储函数是一组为了完成特定功能的 SQL 语句集合。
# 1 存储过程
## 1.1 创建及调用存储过程
```sql
DELIMITER // -- 因为 MySQL 中以分号作为语句的结束，所以需要先将分隔符定义为其他符号，避免 MySQL 处理时遇到过程体中的分号，将它当作结束符了
CREATE PROCEDURE my_proc(IN _name VARCHAR(255)) -- 指定存储过程名、过程参数
BEGIN
	SELECT * FROM `user` WHERE `name`=_name; -- 过程体
END //
DELIMITER ;

CALL my_proc("Leo"); -- 调用存储过程
```
## 1.2 过程参数
过程参数可以分为三类：

- IN：输入参数，调用时可以传递一个变量或值，过程体内只能读取该参数，不能写入。
- OUT：输出参数，调用时只能传递一个变量，过程体内只能写入该参数，不能读取，可以用作返回值。
- INOUT：输入/输出参数，调用时只能传递变量，过程体内可以读取、写入该参数。

注意过程参数的名称不要和列名相同，否则在 SQL 语句中使用时会被当作列名。
## 1.3 变量的使用
存储过程和函数内都可以定义变量，变量的作用域在 BEGIN...END 内：
```sql
DECLARE _name INT DEFAULT 10; -- 定义名为 _name 的 INT 类型变量，默认值 10
SET _name = 20; -- 给变量赋值
SELECT _name; -- 读取变量的值
```
注意，DECLARE 声明一定要放在过程体的最前面，无论是用它声明变量、游标、条件或是 handler，并且要按照变量、游标、条件、handler 的顺序声明。<br />另外，对于 OUT、INOUT 参数，调用时需要传递一个变量，这时就可以在存储过程外创建用户会话变量：
```sql
SET @a = 1; -- 以 @ 开头定义用户会话变量，只在当前会话中有效
```
## **1.4 控制语句**
### 1.4.1 IF
```sql
IF age>20 THEN SET @count1=@count1+1;
	ELSEIF age=20 THEN @count2=@count2+1;
	ELSE @count3=@count3+1;
END lF;
```
### 1.4.2 CASE
```sql
CASE age
    WHEN 20 THEN SET @count1=@count1+1;
    ELSE SET @count2=@count2+1;
END CASE;
```
或者：
```sql
CASE
    WHEN age=20 THEN SET @count1=@count1+1;
    ELSE SET @count2=@count2+1;
END CASE;
```
### 1.4.3 LOOP、LEAVE、ITERATE
LEAVE 相当于编程中的 break，ITERATE 相当于 continue。
```sql
add_num:LOOP
    SET @count=@count+1;
    IF @count=100 THEN
        LEAVE add_num;
    ELSE IF MOD(@count,3)=0 THEN
        ITERATE add_num;
    SELECT * FROM employee;
END LOOP add_num;
```
该示例循环执行 count 加 1 的操作，count 值为 100 时结束循环。如果 count 的值能够整除 3，则跳出本次循环，不再执行下面的 SELECT 语句，进入下一次循环。
### 1.4.4 REPEAT
```sql
REPEAT
    SET @count=@count+1;
UNTIL @count=100
END REPEAT;
```
### 1.4.5 WHILE
```sql
WHILE @count<100 DO
    SET @count=@count+1;
END WHILE;
```
## 1.5 查找存储过程
存储过程的信息保存在 information_schema.Routines 表中：
```sql
SELECT * FROM information_schema.Routines WHERE ROUTINE_NAME LIKE "%my_proc%";
```
## 1.6 查看存储过程状态
```sql
SHOW PROCEDURE STATUS LIKE "%my_proc%";
```
## 1.7 查看存储过程的创建语句
```sql
SHOW CREATE PROCEDURE my_proc;
```
## 1.8 删除存储过程
```sql
DROP PROCEDURE IF EXISTS my_proc; -- 如果存储过程 my_proc 存在，则删除
```
## 1.9 错误处理（条件和 handler）
handler 用于进行错误处理，语法如下：
```sql
DECLARE <handler_type> HANDLER FOR <condition_value> <statement>; -- 当发生指定错误时，statement 指定的 SQL 语句便会执行
```
handler_type 表示错误的处理方式：

- CONTINUE：handler 处理完错误后继续向下执行；
- EXIT：handler 处理完错误后便退出存储过程或存储函数。

condition_value 表示错误的类型：

- sqlstate_value：包含 5 个字符的字符串错误值。
- mysql_error_code：匹配数值类型错误代码。
- condition_name：表示 DECLARE 定义的错误条件名称，定义错误条件的语法如下所示：
```sql
DECLARE can_not_find CONDITION FOR SQLSTATE '42S02'; -- 错误条件名称为 can_not_find，表示 sqlstate_value 值为 42S02
DECLARE can_not_find CONDITION FOR 1146; -- 表示 mysql_error_code 值为 1146
```

- SQLWARNING：匹配所有以 01 开头的 sqlstate_value 值。
- NOT FOUND：匹配所有以 02 开头的 sqlstate_value 值。
- SQLEXCEPTION：匹配所有没有被 SQLWARNING 或 NOT FOUND 捕获的 sqlstate_value 值。

比如，访问一个不存的表，它的错误代码为 1146，定义一个 handler 来处理：
```sql
DELIMITER //
CREATE PROCEDURE my_proc()
BEGIN
	DECLARE CONTINUE HANDLER FOR 1146 UPDATE `user` SET `name`="Leo1"; -- 当发生 1146 错误时，便会执行 UPDATE `user` SET `name`="Leo1"
	SELECT * FROM s; -- s 表不存在，发生 1146 错误
	UPDATE `user` SET `name`="Leo2"; -- 由于 handler 是 CONTINUE 类型，handler 处理完上一条语句的错误后，会继续向下执行，因此这条语句会执行
END //
DELIMITER ;

CALL my_proc(); -- 最终，将 user 表所有记录的 name 列值设为 Leo2
```
注意，DECLARE 声明一定要放在过程体的最前面，无论是用它声明变量、游标、条件或是 handler，并且要按照变量、游标、条件、handler 的顺序声明。
## 1.10 游标
游标用于遍历 SELECT 结果集，只能在存储过程和存储函数中使用。<br />如下所示，使用游标遍历结果集的每一行，并显示每一行的 name 列值：
```sql
DELIMITER //
CREATE PROCEDURE my_proc ()
BEGIN
	DECLARE done INT DEFAULT 0; -- 定义一个循环控制变量
	DECLARE _name VARCHAR(255);

	DECLARE cur CURSOR FOR SELECT `name` FROM `user`; -- 创建游标

	DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET done=1; -- 结果集遍历完了时，再执行 FETCH 会发生 02000 错误，这里处理该错误，将 done 设为 1，以退出后面的循环

	OPEN cur; -- 开启游标

	REPEAT
		FETCH cur INTO _name; -- 访问下一行数据，并存入 _name 变量中
		SELECT _name;
	UNTIL done=1 -- 当 done=1 时终止循环
	END REPEAT;

	CLOSE cur; -- 关闭游标
END //
DELIMITER ;
```
## 1.11 characteristic
创建存储过程时，可以定义 characteristic，其中有用的只有 SQL SECURITY [DEFINER | INVOKER]、COMMENT ''。

1. SQL SECURITY DEFINER：默认值，当调用存储过程的用户 A，具有执行存储过程的权限，创建存储过程的用户 B 具有执行过程体中各 SQL 语句的权限，那么用户 A 就可以执行存储过程。
2. SQL SECURITY INVOKER：调用存储过程的用户 A，必须同时具有执行存储过程的权限、执行过程体中各 SQL 语句的权限，用户 A 才能执行存储过程。
```sql
-- 1. root 用户
DELIMITER //
CREATE PROCEDURE my_proc()
SQL SECURITY INVOKER -- 设为 INVOKER，则调用该存储过程的用户，需要同时具有执行存储过程、执行过程体内 SQL 语句的权限
COMMENT '测试' -- 添加注释
BEGIN
	SELECT * FROM `user`; -- 过程体内对 user 表进行查询
END //
DELIMITER ;

GRANT EXECUTE ON PROCEDURE my_proc TO 'normal_user'@'localhost'; -- 授予用户 normal_user 执行存储过程，即执行 CALL 语句的权限
GRANT SELECT ON test.`user` TO 'normal_user'@'localhost'; -- 授予用户 normal_user 对 user 表的 SELECT 权限

-- 2. normal_user 用户
CALL my_proc(); -- 具有了上面两个权限之后，就可以调用。而如果是 SQL SECURITY DEFINER，因为 root 用户已经有了第二个权限，所以 normal_user 只需要第一个权限即可
```
在存储过程创建之后，还可以再修改 characteristic：
```sql
ALTER PROCEDURE my_proc SQL SECURITY DEFINER COMMENT "test";
```
# 2 存储函数
存储函数与存储过程相比，可以通过 RETURN 来返回值，其他基本都是一样的，只需要把操作存储过程的语句中的 PROCEDURE 替换为 FUNCTION 即可。
```sql
DELIMITER //
CREATE FUNCTION my_fun(_name VARCHAR(255)) -- 参数总是 IN 类型， 没有 OUT、INOUT
RETURNS INT(3) -- 定义返回值类型
BEGIN
	RETURN (SELECT COUNT(*) FROM `user` WHERE `name`=_name); -- 返回值，这里返回 name 列值等于参数 _name 的值的记录的个数
END //
DELIMITER ;

SElECT my_fun("Leo"); -- 调用使用 SELECT
```
