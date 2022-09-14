---
title: MySQL开启远程连接
date: 2018-11-18
categories:
 - 数据库
tags:
 - SQL
 - 数据库
---

## 1 登录MySQL

```bash
mysql -uroot -ppwd
```

## 2 查看user表

```bash
mysql> use mysql
Database changed
mysql> select host,user,password from user;
+------+------+-------------------------------------------+
| host | user | password |
+------+------+-------------------------------------------+
| localhost    | root | *826960FA9CC8A87953B3156951F3634A80BF9853 |
+------+------+-------------------------------------------+
1 row in set (0.00 sec)
```

表中host、user字段标识了可以访问数据库的主机和用户。例如上面的数据就表示只能本地主机通过root用户访问。原来如此，难怪远程连接死活连不上。

## 3 开启远程访问

为了让数据库支持远程主机访问，有两种方法可以开启远程访问功能。

### 3.1 改表法

修改host字段的值，将localhost修改成需要远程连接数据库的ip地址。或者直接修改成%。修改成%表示，所有主机都可以通过root用户访问数据库。为了方便，我直接修改成%。命令：

```bash
mysql> update user set host = '%' where user = 'root';
```

再次查看user表

```bash
+------+------+-------------------------------------------+
| host | user | password |
+------+------+-------------------------------------------+
| % | root | *826960FA9CC8A87953B3156951F3634A80BF9853 |
+------+------+-------------------------------------------+
1 row in set (0.00 sec)
```

修改成功，输入以下命令，回车使刚才的修改生效，再次远程连接数据库成功。

```bash
mysql> FLUSH PRIVILEGES;
```

### 3.2 授权法

例如，你想root使用mypassword从任何主机连接到mysql服务器的话： 

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'mypassword' WITH GRANT OPTION; 
```

如果你想允许用户myuser从ip为 `192.168.1.3` 的主机连接到mysql服务器，并使用mypassword作为密码：

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.1.3' IDENTIFIED BY 'mypassword' WITH GRANT OPTION; 
```

### 3.3 创建用户法

先创建一个用户名为username，密码为password，远程权限为任何ip（%）的用户：

```sql
create user username@'%' identified by 'password';
```

授予username管理所有数据库的全部权限：

```sql
grant all privileges on *.* to username;
```

输入以下命令，回车使刚才的修改生效，再次远程连接数据库成功。

```bash
mysql> FLUSH PRIVILEGES;
```

别忘记最后的 `FLUSH PRIVILEGES;` 刷新先前的修改。
