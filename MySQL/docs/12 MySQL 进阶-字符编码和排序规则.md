# 1 字符串类型的存储
默认情况下，MySQL 使用 utf8 字符编码和 utf8_general_ci 排序规则来存储字符串类型。<br />但是最好使用 utf8mb4，原因在于：

- MySQL 中的 utf8 并不是真正意义上的 utf8，真正的 utf8 最多占用 4 个字节，而 MySQL 中的 utf8 最多占用 3 个字节，这就导致一些字符，比如 emoji 字符无法存储。
- MySQL 中的 utf8mb4 才是真正意义上的 utf8，最多占用 4 个字节，可以存储 emoji 等 4 字节字符，并且 utf8mb4 对 utf8 是兼容的，可以直接将 utf8 切换为 utf8mb4。

排序规则是用于比较字符大小的，可以设置区分或不区分大小写：

- utf8mb4_general_ci：当字符编码为 utf8mb4 时的默认值，不区分大小写，比如 A 和 a 是一样的。
- utf8mb4_bin：区分大小写，它是直接按字节比较。

MySQL 提供了两个系统变量来设置字符串类型存储时使用的字符编码和排序规则：

- character_set_server：字符编码。
- collation_server：排序规则。

也可以在配置文件中设置：
```bash
[mysqld]
character_set_server=utf8mb4
collation_server=utf8mb4_general_ci
```
# 2 客户端和 MySQL 服务器的通信
客户端与 MySQL 服务器之间发送的数据其实就是一个字节串，如果列为字符串类型，在通信过程中会进行多次字符编码的转换，其中涉及三个系统变量：

1. MySQL 首先按照 character_set_client 编码将请求中的字节串解码为字符串。
2. 将上一步得到的字符串按照 character_set_connection 编码为字节串。
3. 将上一步得到的字节串再按照 character_set_connection 重新解码为字符串，再按照列的编码来编码得到字节串，然后存储。
4. MySQL 向客户端返回数据时，将数据按照列的编码进行解码，得到字符串，再将这个字符串用 character_set_results 编码为字节串，然后将它返回给客户端。

一般将这三个系统变量设置成和客户端使用的字符编码、以及列的编码一致，可以在配置文件中设置这三个系统变量的值：
```bash
[mysql]
default-character-set=utf8mb4
```
