---
title: Docker部署MySQL主从
date: 2021-04-15 21:27:45
tags: db

---

一、安装基础环境及mysql

```shell
1.安装docker-compose
  curl -L "https://github.com/docker/compose/releases/download/1.29.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  
  chmod +x /usr/local/bin/docker-compose
  
2.编写docker-compose

version: '3.8'
services:
  mysql-master:
    container_name: mysql-master 
    image: mysql:5.7.31
    restart: always
    ports:
      - 3340:3306 
    privileged: true
    volumes:
      - $PWD/msql-master/volumes/log:/var/log/mysql  
      - $PWD/msql-master/volumes/conf/my.cnf:/etc/mysql/my.cnf
      - $PWD/msql-master/volumes/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb
      
  mysql-slave:
    container_name: mysql-slave 
    image: mysql:5.7.31
    restart: always
    ports:
      - 3341:3306 
    privileged: true
    volumes:
      - $PWD/msql-slave/volumes/log:/var/log/mysql  
      - $PWD/msql-slave/volumes/conf/my.cnf:/etc/mysql/my.cnf
      - $PWD/msql-slave/volumes/data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb    

networks:

  myweb:
    driver: bridge
    
3.创建配置文件目录
  mkdir -p msql-master/volumes/conf
  mkdir -p msql-slave/volumes/conf
  
  [root@VM-0-2-centos mysqlMS]# tree 
    .
    |-- msql-master
    |   `-- volumes
    |       `-- conf
    `-- msql-slave
        `-- volumes
            `-- conf
            
            
4.创建mysql配置文件

主库配置文件
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段
server-id=1

# [必须]启用二进制日志
log-bin=mysql-bin 

# 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql

# 设置需要同步的数据库 binlog_do_db = 数据库名； 
# 如果是多个同步库，就以此格式另写几行即可。
# 如果不指明对某个具体库同步，表示同步所有库。除了binlog-ignore-db设置的忽略的库
# binlog_do_db = test #需要同步test数据库。

# 确保binlog日志写入后与硬盘同步
sync_binlog = 1

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all    

​```
温馨提示：在主服务器上最重要的二进制日志设置是sync_binlog，这使得mysql在每次提交事务的时候把二进制日志的内容同步到磁盘上，即使服务器崩溃也会把事件写入日志中。
sync_binlog这个参数是对于MySQL系统来说是至关重要的，他不仅影响到Binlog对MySQL所带来的性能损耗，而且还影响到MySQL中数据的完整性。对于``"sync_binlog"``参数的各种设置的说明如下：
sync_binlog=0，当事务提交之后，MySQL不做fsync之类的磁盘同步指令刷新binlog_cache中的信息到磁盘，而让Filesystem自行决定什么时候来做同步，或者cache满了之后才同步到磁盘。
sync_binlog=n，当每进行n次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘。
  
在MySQL中系统默认的设置是sync_binlog=0，也就是不做任何强制性的磁盘刷新指令，这时候的性能是最好的，但是风险也是最大的。因为一旦系统Crash，在binlog_cache中的所有binlog信息都会被丢失。而当设置为“1”的时候，是最安全但是性能损耗最大的设置。因为当设置为1的时候，即使系统Crash，也最多丢失binlog_cache中未完成的一个事务，对实际数据没有任何实质性影响。
  
从以往经验和相关测试来看，对于高并发事务的系统来说，“sync_binlog”设置为0和设置为1的系统写入性能差距可能高达5倍甚至更多。
​```

备库配置文件
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段  
server-id=2

# 如果想实现 主-从（主）-从 这样的链条式结构，需要设置：
# log-slave-updates      只有加上它，从前一台机器上同步过来的数据才能同步到下一台机器。

# 设置需要同步的数据库，主服务器上不限定数据库，在从服务器上限定replicate-do-db = 数据库名；
# 如果不指明同步哪些库，就去掉这行，表示所有库的同步（除了ignore忽略的库）。
# replicate-do-db = test；

# 不同步test数据库 可以写多个例如 binlog-ignore-db = mysql,information_schema 
replicate-ignore-db=mysql  

## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-bin
log-bin-index=mysql-bin.index

## relay_log配置中继日志
#relay_log=edu-mysql-relay-bin  

## 还可以设置一个log保存周期：
#expire_logs_days=14

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all  
```

二、启动mysql主从并查看ip

```
docker-compose up -d
Creating network "mysqlms_myweb" with driver "bridge"
Creating mysql-master ... done
Creating mysql-slave  ... done
```

```
[root@VM-0-2-centos conf]# docker network inspect mysqlms_myweb|grep -A 4 IPv4Address
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "af8bab735166707d64bdba3591fc2ad7d8deaaff9b2e165ca3ebc03a94f84197": {
                "Name": "mysql-slave",
--
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},

```



三、配置复制用户并启动同步

```
主库操作
1.查看主库日志位置
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      154 |              | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

2.配置复制用户
mysql> grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
mysql> flush privileges;

备库操作
1.配置备库
change master to master_host='192.168.112.3',master_user='slave',master_password='123456',master_port=3306,master_log_file='mysql-bin.000005', master_log_pos=154,master_connect_retry=30;

​```
**master_port**：Master的端口号，指的是容器的端口号

**master_user**：用于数据同步的用户

**master_password**：用于同步的用户的密码

**master_log_file**：指定 Slave 从哪个日志文件开始复制数据，即上文中提到的 File 字段的值

**master_log_pos**：从哪个 Position 开始读，即上文中提到的 Position 字段的值

**master_connect_retry**：如果连接失败，重试的时间间隔，单位是秒，默认是60秒
​```

2.启动备库同步
mysql> start slave;
mysql> show slave status \G;
```





