---
layout: post
title: "ubuntu安装MySQL及配置"
date: 2019-11-23
excerpt: 
tags: [Linux, mysql]
feature: 
comments: false
---

之前一直用的Cent OS，第一次用Ubuntu做服务器装MySQL，记录一下。

### 环境

Ubuntu18.04LTS（腾讯云的一台机器）

### 安装

首先到[MySQL官方的apt仓库](https://dev.mysql.com/downloads/repo/apt/)选一个版本的deb包，这里只有一个，但是只能用图形化方式下载，而云服务器上无法用这种方式，在Windows上用浏览器下载了之后，在下载记录里找到了下载的具体地址（https://repo.mysql.com//mysql-apt-config_0.8.14-1_all.deb），于是在Ubuntu上第一步先下载一个deb包

```
wget https://repo.mysql.com//mysql-apt-config_0.8.14-1_all.deb
```

下载到当前目录，安装下载的包

```bash
sudo dpkg -i mysql-apt-config_0.8.14-1_all.deb
```

会弹出一个比较图形化的配置界面

![](https://dle.oss-cn-beijing.aliyuncs.com/18-7-21/%E6%89%B9%E6%B3%A8%202019-11-21%20174134.png)

<img src='https://dle.oss-cn-beijing.aliyuncs.com/18-7-21/%E6%89%B9%E6%B3%A8%202019-11-21%20174134.png' align=center width = 80% height = 80%>

在第一个里面选择版本，比如5.7，然后选择OK。

然后更新包信息，安装MySQL

```bash
sudo apt-get update
sudo apt-get install mysql-server
```

安装过程需要下载一些东西，可能会比较**慢**，可以试试先找个国内的镜像源（有些真不知道有没有）换上，安装的最后会弹出一个界面输入root用户的密码，也可以不设置留为空。安装完成后可以查看状态、开启关闭服务等

```bash
# 查看状态
service mysql status
# 关闭服务
service mysql stop
# 开启服务
service mysql start
# 重启服务
service mysql restart
```

安装和另外一些东西可以参考[MySQL官方安装文档](https://dev.mysql.com/doc/mysql-apt-repo-quick-guide/en/#apt-repo-fresh-install)

### 登录

```bash
mysql -u root -p
```

然后输入刚才设置的密码就可以登录root用户了，登录成功后命令行会变成`mysql>`这样，可以使用MySQL语句进行操作。

### 配置

MySQL的配置文件位于`/etc/mysql/`下面`my.cnf`，详细的配置项可以查看有关服务器系统变量的[文档](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html)。

#### 更改字符集

MySQL默认的字符集是Latin-1，存中文会乱码，换成utf-8能满足需求，这就需要修改配置文件

```bash
vi /etc/mysql/my.cnf
```

之前Cent OS上的配置文件都是比较满的，Ubuntu上几乎什么都没有，所以直接在后面添加这么几行

```
[client]
default_character_set=utf8mb4

[mysql]
default_character_set=utf8mb4

[mysqld]
character_set_server=utf8mb4
```

保存后，重启MySQL的服务，然后可以登录MySQL，用下面的命令查看字符集

```mysql
mysql> show variables like '%char%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8mb4                    |
| character_set_connection | utf8mb4                    |
| character_set_database   | utf8mb4                    |
| character_set_filesystem | binary                     |
| character_set_results    | utf8mb4                    |
| character_set_server     | utf8mb4                    |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

#### 开启日志

二进制日志有助于在不小心造成了某些意外的操作比如删库这种情况下帮助恢复数据，当然及时的**备份**（冷备、热备）也是十分有必要的。

同样地，编辑配置文件，在my.cnf中的[mysqld]下面添加以下内容

```
# id在集群中需要唯一，如果只有一个库，随便一个都可以
server-id = 1
# 二进制日志的存放位置
log-bin = /var/log/mysql/mysql-bin.log
log-bin-index = binlog.index
max_binlog_size = 1G
binlog_format = row
binlog_row_image = full

# 顺便把日志一起开了
general_log_file = /var/log/mysql/mysql.log
general_log = 1
```

改完以后不要忘了重启服务。

[备份](https://www.cnblogs.com/kissdodog/p/4174421.html)和[恢复](https://www.jianshu.com/p/c9a2fe3f4534)的问题暂时还没有遇到，可以参考这两个链接。