+++
title = "Config Redis Replication on Centos 5"
date = "2013-06-25"
slug = "2013/06/25/config-redis-replication-on-centos-5"
Categories = []
+++

## 安装 Redis 2.6.14

* CentOS 5.4
* Redis 2.6.14

下载并安装：

```bash
$ wget http://redis.googlecode.com/files/redis-2.6.14.tar.gz
$ cd redis-2.6.14
$ make
$ sudo make install
```

CentOS 5自带的Tcl版本是8.4，Redis的`make test`需要用到Tcl 8.5以上的版本，如果你需要运行测试用例，请先安装新版本的Tcl，然后再安装Redis：

```bash
$ wget http://prdownloads.sourceforge.net/tcl/tcl8.6.0-src.tar.gz
$ tar xzvf tcl8.6.0-src.tar.gz
$ cd tcl8.6.0/unix
$ ./configure
$ make
$ make test
$ sudo make install
```

## 配置 Redis 服务

### 修复安装脚本 utils/install_server.sh 的问题

这个版本的安装脚本有一个小问题，请看 [#909](https://github.com/antirez/redis/pull/909) Pull Request:

```diff
@@ -159,7 +159,7 @@ REDIS_CHKCONFIG_INFO=\
 # Description: Redis daemon\n
 ### END INIT INFO\n\n"
 
-if [ !`which chkconfig` ] ; then 
+if [ ! `which chkconfig` ] ; then 
   #combine the header and the template (which is actually a static footer)
   echo $REDIS_INIT_HEADER > $TMP_FILE && cat $INIT_TPL_FILE >> $TMP_FILE || die "Could not write init script to $TMP_FILE"
 else
@@ -173,7 +173,7 @@ echo "Copied $TMP_FILE => $INIT_SCRIPT_DEST"
 
 #Install the service
 echo "Installing service..."
-if [ !`which chkconfig` ] ; then 
+if [ ! `which chkconfig` ] ; then 
   #if we're not a chkconfig box assume we're able to use update-rc.d
   update-rc.d redis_$REDIS_PORT defaults && echo "Success!"
 else
```

### 修正sudo的时候的环境变量 $PATH 的问题

默认的情况下，使用`sudo`命令时的`$PATH`环境变量只有`/usr/bin:/bin`两个路径，但是`install_server.sh`需要更多的路径：

* `chkconfig`在`/sbin`目录下面
* `redis-server`和`redis-cli`默认安装在`/usr/local/bin`下面

所以我们必须修改`sudo`命令时的环境变量。我喜欢使用的方式是做一个alias，修改`~/.bashrc`文件：

```bash
alias sudo='sudo env PATH=/usr/bin:/bin:/sbin:/usr/sbin:/usr/local/bin'
```

重新登录以后就可以进行Redis服务的配置了！

### 配置 Master-Slave Replication

首先配置两个Redis实例，分别使用端口`6379`和`6380`，把目录安装在`/data0/redis/6379`和`/data0/redis/6380`下面：

```bash
$ cd utils
$ sudo ./install_server.sh
Password:

Welcome to the redis service installer
This script will help you easily set up a running redis server

Please select the redis port for this instance: [6379]
Selecting default: 6379
Please select the redis config file name [/etc/redis/6379.conf]
Selected default - /etc/redis/6379.conf
Please select the redis log file name [/var/log/redis_6379.log]
Selected default - /var/log/redis_6379.log
Please select the data directory for this instance [/var/lib/redis/6379] /data0/redis/6379
Selected default - /var/lib/redis/6379
Please select the redis executable path [/usr/local/bin/redis-server]
s#^port [0-9]{4}$#port 6379#;s#^logfile .+$#logfile /var/log/redis_6379.log#;s#^dir .+$#dir /var/lib/redis/6379#;s#^pidfile .+$#pidfile /var/run/redis_6379.pid#;s#^daemonize no$#daemonize yes#;
Copied /tmp/6379.conf => /etc/init.d/redis_6379
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!

$ sudo ./install_server.sh

Welcome to the redis service installer
This script will help you easily set up a running redis server


Please select the redis port for this instance: [6379] 6380
Please select the redis config file name [/etc/redis/6380.conf]
Selected default - /etc/redis/6380.conf
Please select the redis log file name [/var/log/redis_6380.log]
Selected default - /var/log/redis_6380.log
Please select the data directory for this instance [/var/lib/redis/6380] /data0/redis/6380
Selected default - /var/lib/redis/6380
Please select the redis executable path [/usr/local/bin/redis-server]
s#^port [0-9]{4}$#port 6380#;s#^logfile .+$#logfile /var/log/redis_6380.log#;s#^dir .+$#dir /var/lib/redis/6380#;s#^pidfile .+$#pidfile /var/run/redis_6380.pid#;s#^daemonize no$#daemonize yes#;
Copied /tmp/6380.conf => /etc/init.d/redis_6380
Installing service...
Successfully added to chkconfig!
Successfully added to runlevels 345!
Starting Redis server...
Installation successful!
```

加入我们把 6379 作为 Master，把 6380 作为 Slave，我们只需要修改`/etc/redis/6380.conf`：

```
slaveof 127.0.0.1 6379
```

重启Slave即可完成一个Master-Slave集群的配置：

```bash
$ sudo service redis_6380 stop
$ sudo service redis_6380 start
```

### 测试 Master-Slave 集群

首先，连接到 Master 并写入数据：

```bash
$ redis-cli -p 6379
redis 127.0.0.1:6379> set hover-name Ruan
OK
```

然后连接到 Slave 测试是否能读取到数据，并尝试在 Slave 写数据：

```bash
$ redis-cli -p 6380
redis 127.0.0.1:6380> get hover-name
"Ruan"
redis 127.0.0.1:6380> set hover-name Hover
(error) READONLY You can't write against a read only slave.
```

OK，Master-Slave Replication 算是配置成功了！

