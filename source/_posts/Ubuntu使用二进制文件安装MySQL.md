---
title: Ubuntu使用二进制文件安装MySQL
tags:
  - MySQL
date: 2024-12-15 15:44:55
---


- [1. 下载](#1-下载)
- [2. 安装](#2-安装)
  - [2.1. 初始化数据目录](#21-初始化数据目录)
  - [2.2. 使用systemed管理MysQL服务](#22-使用systemed管理mysql服务)
- [3. support-files/mysql.server脚本文件](#3-support-filesmysqlserver脚本文件)
- [4. 参考文章](#4-参考文章)


# 1. 下载

下载官网：https://dev.mysql.com/downloads/mysql/

{% asset_img "MySQL下载.png" "MySQL下载" %}

查看glibc版本
```shell
whn@study:~$ /lib/x86_64-linux-gnu/libc.so.6
GNU C Library (Ubuntu GLIBC 2.31-0ubuntu9.16) stable release version 2.31.
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 9.4.0.
libc ABIs: UNIQUE IFUNC ABSOLUTE
For bug reporting instructions, please see:
<https://bugs.launchpad.net/ubuntu/+source/glibc/+bugs>.

```

可以看到glibc版本为2.31，查看系统的详细信息可以使用`uname -a`：

```shell
whn@study:~$ uname -a
Linux study 5.4.0-200-generic #220-Ubuntu SMP Fri Sep 27 13:19:16 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```


# 2. 安装

## 2.1. 初始化数据目录

解压到`/usr/local/`并为其创建软链接

```shell
# 解压xz文件使用 -J
sudo tar -Jxvf mysql-8.0.40-linux-glibc2.28-x86_64.tar.xz -C /usr/local/
# 创建软链接
cd /usr/local/
sudo ln -s mysql-8.0.40-linux-glibc2.28-x86_64 mysql
```

MySQL文件列表如下：

{% asset_img "MySQL文件.png" "MySQL文件" %}

|目录|内容|
|----|---|
|bin|可执行文件|
|docs|文档|
|man|unix手册|
|include|包含的文件|
|lib|依赖库|
|share|错误信息、字典和数据库安装 SQL|
|support-files|支持文件|

为了更方便的使用命令，为MySQL配置环境变量(全局)，`sudo vim /etc/profile.d/global.sh`

```shell
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
```

增加可执行权限，更新环境变量并查看

```shell
whn@study:~$ sudo chmod +x /etc/profile.d/global.sh 
whn@study:~$ source /etc/profile
whn@study:~$ echo $MYSQL_HOME
/usr/local/mysql
whn@study:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/local/mysql/bin
```


MySQL 依赖于libaio，如果没安装这个库，数据目录初始化和后续服务器启动步骤将会失败。

查看是否已经安装
```shell
whn@study:/usr/local/mysql$ dpkg -l | grep libaio
ii  libaio1:amd64                         0.3.112-5                         amd64        Linux kernel AIO access library - shared library
```

如果没有libaio，则需要进行安装

```shell
# 查找可用软件包
whn@study:/usr/local/mysql$ apt-cache search libaio
libaio-dev - Linux kernel AIO access library - development files
libaio1 - Linux kernel AIO access library - shared library
# 发现有两个包，选择libaio1
whn@study:/usr/local/mysql$ sudo apt install libaio1
```


创建mysql用户组以及用户

```shell
# 创建mysql用户组，-r表示是系统组
sudo groupadd -r mysql
# 创建mysql系统用户并为其指定组
sudo useradd -M -N -g mysql -r -d /usr/local/mysql/data -s /bin/false -c "MySQL Server" mysql
```

创建用户的参数说明：

- -M：不创建家目录
- -N：不创建与用户同名的组，而是将用户添加到-g指定的组中
- -g：指定所属组
- -r：系统用户
- -d：用户登陆使用的目录，这里设置为MySQL的数据目录
- -s：登陆使用shell
- -c：备注

拥有**FILE**权限的MySQL用户有权使用`LOAD DATA INFILE`和`SELECT ... INTO OUTFILE`语句以及`LOAD_FILE()`函数读写服务器的文件。

默认情况下，拥有**FILE**权限的用户可以读取服务器全部可读或MySQL服务器可读的任何文件(这意味着用户可以读取任何数据库目录下的任何文件，因为服务器可以访问这些文件中的任何一个）。**FILE**权限还允许用户在MySQL服务器有写权限的任何目录中创建新文件，包括服务器数据目录，其中包含实现权限表的文件。这会影响系统的安全性，我们需要对其能够操作的目录范围进行限制。
 
要限制**FILE**权限的范围，需要创建一个拥有**FILE**权限的用户可安全用于导入和导出操作的目录。 本篇文章中，创建的目录名为`mysql-files`，位于数据目录下。 在以后配置服务器启动选项时，将`secure_file_priv`选项设置为`mysql-files`目录。

```shell
cd /usr/local/mysql
sudo mkdir mysql-files
sudo chown mysql:mysql mysql-files
sudo chmod 750 mysql-files
```

创建配置文件`/etc/my.cnf`：

```shell
cd /etc
sudo touch my.cnf
sudo chown root:root my.cnf
sudo chmod 644 my.cnf
```

这里只配置MySQL的基础选项，其他可以根据实际情况自行配置

```
[mysqld]
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
port=3306
user=mysql
secure_file_priv=/usr/local/mysql/mysql-files
local_infile=OFF
```

- basedir：安装目录
- datadir：数据目录
- port：端口
- user：设置用户选项，确保服务器以非特权的 mysql 用户账户启动。 出于安全考虑，必须避免以操作系统 root 用户身份运行服务器
- secure_file_priv：FILE权限可操作的目录，如果想要禁止导入、导出可设置为`NULL`
- local_infile：避免LOCAL版本的`LOAD DATA`的[安全问题](https://dev.mysql.com/doc/refman/8.0/en/load-data-local-security.html)，将其禁用



初始化数据目录，这里我们可以不手动创建目录，数据初始化的时候会创建

```shell
whn@study:/usr/local/mysql$ sudo bin/mysqld --defaults-file=/etc/my.cnf --initialize
2024-12-15T04:20:41.373108Z 0 [System] [MY-013169] [Server] /usr/local/mysql-8.0.40-linux-glibc2.28-x86_64/bin/mysqld (mysqld 8.0.40) initializing of server in progress as process 8550
2024-12-15T04:20:41.381792Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2024-12-15T04:20:41.745441Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2024-12-15T04:20:45.648431Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: ,oQ!fmS2t(ri
```

生成的随机密码为`,oQ!fmS2t(ri`


## 2.2. 使用systemed管理MysQL服务

添加`mysqld.service`文件

```shell
cd /usr/lib/systemd/system
sudo touch mysqld.service
sudo chmod 644 mysqld.service
```

mysqld.service文件内容如下：

```
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target

[Service]
User=mysql
Group=mysql

# Have mysqld write its state to the systemd notify socket
Type=notify

# Disable service start and stop timeout logic of systemd for mysqld service.
TimeoutSec=0

# Start main service
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf $MYSQLD_OPTS 

# Use this to switch malloc implementation
EnvironmentFile=-/etc/sysconfig/mysql

# Sets open_files_limit
LimitNOFILE=10000

Restart=on-failure

RestartPreventExitStatus=1

# Set environment variable MYSQLD_PARENT_PID. This is required for restart.
Environment=MYSQLD_PARENT_PID=1

PrivateTmp=false
```

设置为开机启动

```shell
sudo systemctl daemon-reload
sudo systemctl enable mysqld.service
```

启动并测试

```shell
whn@study:~$ sudo systemctl start mysqld
whn@study:~$ systemctl status mysqld.service 
● mysqld.service - MySQL Server
     Loaded: loaded (/lib/systemd/system/mysqld.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-12-15 06:33:55 UTC; 1min 26s ago
       Docs: man:mysqld(8)
             http://dev.mysql.com/doc/refman/en/using-systemd.html
   Main PID: 22906 (mysqld)
     Status: "Server is operational"
      Tasks: 37 (limit: 4556)
     Memory: 382.1M
     CGroup: /system.slice/mysqld.service
             └─22906 /usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf
```

使用`mysqlshow`进行测试，密码为初始化目录时生成的随机密码，这里是`,oQ!fmS2t(ri`

```shell
whn@study:~$ mysqlshow -u root -p
Enter password: 
+--------------------+
|     Databases      |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

重置root用户的密码

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'password';
```

# 3. support-files/mysql.server脚本文件

`support-files/mysql.service`脚本也可以用于管理mysql服务，里面调用`mysqld_safe`，`mysql_safe`有调用`mysqld`开启/关闭mysql，我们可以将其复制到`/etc/init.d`作为服务管理

使用该脚本我们需要配置`basedir`、`datadir`路径，不配置就会去`/etc/my.cnf`中获取，说明如下。

如果我们什么都没有配置，直接调用该脚本`mysql.service start`，那么它会默认为安装路径为`/usr/local/mysql`，代码如下：

```shell
if test -z "$basedir"
then
  basedir=/usr/local/mysql
  bindir=/usr/local/mysql/bin
  if test -z "$datadir"
  then
    datadir=/usr/local/mysql/data
  fi
  sbindir=/usr/local/mysql/bin
  libexecdir=/usr/local/mysql/bin
else
  bindir="$basedir/bin"
  if test -z "$datadir"
  then
    datadir="$basedir/data"
  fi
  sbindir="$basedir/sbin"
  libexecdir="$basedir/libexec"
fi
```

我们如果没有在`/usr/local/mysql`安装，那么后面会去`/etc/my.cnf`配置文件中去找`basedir`配置，然后去调用`my_print_defaults`获取所有配置文件的配置项，获取`basedir`配置代码如下：

```shell
if test -x "$bindir/my_print_defaults";  then
  print_defaults="$bindir/my_print_defaults"
else
  # Try to find basedir in /etc/my.cnf
  conf=/etc/my.cnf
  print_defaults=
  if test -r $conf
  then
    subpat='^[^=]*basedir[^=]*=\(.*\)$'
    dirs=`sed -e "/$subpat/!d" -e 's//\1/' $conf`
    for d in $dirs
    do
      d=`echo $d | sed -e 's/[  ]//g'`
      if test -x "$d/bin/my_print_defaults"
      then
        print_defaults="$d/bin/my_print_defaults"
        break
      fi
    done
  fi

  # Hope it's in the PATH ... but I doubt it
  test -z "$print_defaults" && print_defaults="my_print_defaults"
fi
```

转换参数代码如下：

```shell
extra_args=""
if test -r "$basedir/my.cnf"
then
  extra_args="-e $basedir/my.cnf"
fi

parse_server_arguments `$print_defaults $extra_args mysqld server mysql_server mysql.server`
```

配置文件路径如下：

|文件名|说明|
|-----|----|
|/etc/my.cnf|全局配置|
|/etc/mysql/my.cnf|全局配置|
|SYSCONFDIR/my.cnf|全局配置|
|$MYSQL_HOME/my.cnf|Server-specific options (server only)|
|defaults-extra-file|The file specified with --defaults-extra-file, if any|
|~/.my.cnf|用户特定配置|
|~/.mylogin.cnf|User-specific login path options (clients only)|
|DATADIR/mysqld-auto.cnf|System variables persisted with SET PERSIST or SET PERSIST_ONLY (server only)|

# 4. 参考文章

安装：https://dev.mysql.com/doc/mysql-secure-deployment-guide/8.0/en/