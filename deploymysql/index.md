# 部署mysql（GTID主从复制）


手上是两台新安装的服务器，系统为CentOS7，mysql的版本为5.7，登录的用户为root

## 1. 改源
把安装的源地址改成阿里云的
```shell
shell> cd /etc/yum.repos.d
shell> mv CentOS-Base.repo CentOS-Base.repo.back # 备份
shell> wget -O CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
shell> yum clean all
shell> yum makecache
```

## 2. 安装

获取mysql的源库，也可以到[mysql官网](https://dev.mysql.com/downloads/repo/yum/)查找其他版本
```shell
# 基于el7系统
shell> wget https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm
```

安装源库
```shell
shell> rpm -Uvh mysql57-community-release-el7-11.noarch.rpm
```
如果之前安装过的话会提示错误
```shell
警告：mysql57-community-release-el7-11.noarch.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
错误：依赖检测失败：
        mysql57-community-release 与 (已安裝) mysql80-community-release-el7-3.noarch 冲突
```
解决方法，或者直接使用这个命令安装源库
```shell
shell> yum install mysql57-community-release-el7-11.noarch.rpm
```

安装mysql
```shell
yum install mysql-community-server
```

提示错误，由于mariadb引起的
```shell
错误：软件包：akonadi-mysql-1.9.2-4.el7.x86_64 (@anaconda)
          需要：mariadb-server
          正在删除: 1:mariadb-server-5.5.64-1.el7.x86_64 (@anaconda)
              mariadb-server = 1:5.5.64-1.el7
          取代，由: mysql-community-server-5.7.33-1.el7.x86_64 (mysql57-community)
              未找到
          更新，由: 1:mariadb-server-5.5.68-1.el7.x86_64 (base)
              mariadb-server = 1:5.5.68-1.el7
```

删除本机上的MariaDB，如果有的话：
```shell
shell> yum list installed mariadb\*
mariadb.x86_64
mariadb-libs.x86_64
mariadb-server.x86_64

shell> yum remove mariadb.x86_64 mariadb-libs.x86_64 mariadb-server.x86_64

# 重新安装即可
shell> yum install mysql-community-server
```


## 3. 启动
启动mysql服务器
```shell
shell> service mysqld start
```

查看一下服务的状态
```shell
shell> service mysqld status
● mysqld.service - MySQL Server
    Active: active (running)
```

首次启动服务，会进行如下操作
- 服务器初始化
- 在数据目录里生成SSL证书和密钥文件
- 已安装并启用[validate_password](https://www.docs4dev.com/docs/zh/mysql/5.7/reference/validate-password.html)
- 创建了一个超级用户帐户`'root'@'localhost'`。生成了一个随机密码，使用如下命令进入日志文件中能查看：
  ```shell
  shell> grep 'temporary password' /var/log/mysqld.log
  ```

使用密码登录root帐户
```shell
shell> mysql -u root -p
```

记得修改密码
```sql
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPassword';
```

**另外：其中一台服务器启动时报错：**
`Unable to determine if daemon is running: No such file or directory`

使用如下命令解决：
```shell
shell> service mysqld stop
shell> rm -rf /var/lib/mysql/*
shell> service mysqld start
```

安装完后可以使用以下命令做一些安全设置
```shell
shell> mysql_secure_installation
Change the password for root ? # 上面已经改了这里不再修改 直接回车下一步
Remove anonymous users? # y
Disallow root login remotely? # y
Remove test database and access to it? # y
Reload privilege tables now?  # y
```

## 4. 复制

*使用全局事务标识符（GTID）复制*

数据库每提交一个事务，都会生成一个标识符，并记录在binlog中，该标识符在源数据库上唯一，且下面所有副本服务器中也是唯一。代替了基于binlog复制的file和pos。但需要在副本数据库中开启binlog，这有意味着在切换新源或启用新副本时不必引用源的binlog，容易保证数据的一致性。如果副本不开启binlog则会存储在table`mysql.gtid_executed`里。

> 注意：必须在源服务器上启用binlog才能进行复制

### 4.1 设置配置

关闭mysql数据库：
```shell
shell> mysqladmin -u root -p shutdown
```

设置配置，配置文件在`/etc/my.conf`。或者使用以下命令查找：
```shell
shell> mysql --verbose --help | grep my.cnf
```

增加源配置：
```shell
[mysqld]
log-bin=mysql-bin  # 启用二进制日志文件 必须
server-id=1        # 设置服务唯一id
gtid_mode=ON       # 启用基于GTID的复制
enforce-gtid-consistency=ON  # 启用GTID模式
```

副本的配置如下：
```shell
[mysqld]
server-id=2
gtid_mode=ON
enforce-gtid-consistency=ON

# 启用日志（可选）
# 启用日志可以在binlog中查找gtid和对应的语句
# 不启用则需要在mysql.gtid_executed表中查找gtid，然后到源上查找对应到语句
# binlog与gtid_executed只生效其中一个
log-bin=mysql-bin
log-slave-updates=ON
```
启动数据库

### 4.2 创建副本用户

在源上创建一个新的帐号，用于给所有的副本连接，并提供复制的权限。
```sql
mysql> CREATE USER 'repl'@'192.168.7.%' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.7.%';
```

### 4.3 配置副本为GTID自动定位

配置副本使用GTID的事务的源来更新数据，并使用GTID定位。

```sql
mysql> CHANGE MASTER TO
    -> MASTER_HOST = '192.168.7.30',
    -> MASTER_PORT = 3306,
    -> MASTER_USER = 'repl',
    -> MASTER_PASSWORD = 'password',
    -> MASTER_AUTO_POSITION = 1;
```

启动副本：
```sql
mysql> START SLAVE;
```

### 4.4 （可选）设置防火墙

如果开启了防火墙则需要开放3306端口：
```shell
shell> firewall-cmd --zone=public --add-port=3306/tcp --permanent
shell> firewall-cmd --reload
```

