# 搭建高可用 mongodb 集群-副本集

### 环境装备

* **三台机器**

> prosvr-6cf7-191010 172.19.30.240  
> prosvr-ea03-191010 172.19.20.187  
> prosvr-dd0e-191010 172.19.40.70

* **关闭 Transparent HugePages**

> 从 rhel6 开始系统将默认对所有程序开启 transparent\_hugepage（THP），linux 内核将尽可能的尝试分配 2M 的页大小给程序使用，内核空间在内存中自身是 2M 对齐的，目的是减少内核 TLB（现代 CPU 使用一小块关联内存，用来缓存最近访问的虚拟页的 PTE。这块内存称为 translation lookaside buffer）的压力，增大 page 大小自然会减少 TLB 大小。如果内存没有 2M 连续大小的空间可分配，内核会回退到分配 4KB 页的方案。THP 页也是可以换出的，这是通过把 2M 的大页分割成 4KB 的普通页实现的。

> 内核通过增加一个 khugepaged 内核线程，来不停的寻找连续的足够大的尽量对其的内存区域来满足内存分配请求。这个线程会偶尔尝试使用大的内存分配来替换连续的小内存页以最大化 THP 的使用。

> ORACLE 和 mongodb 数据库不建议开启 THP，THP 在运行时动态分配内存，可能会带来运行时内存分配的延误。

> 数据库应用不建议开启 Transparent HugePages

查看系统是否开启了 THP 命令

```text
[root@prosvr-6cf7-191010 logs]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never

[root@prosvr-6cf7-191010 logs]# cat /sys/kernel/mm/transparent_hugepage/defrag
always defer defer+madvise [madvise] never
```

实时生效命令，机器重启后会失效

```text
[root@prosvr-6cf7-191010 logs]# echo never >> /sys/kernel/mm/transparent_hugepage/enabled

[root@prosvr-6cf7-191010 logs]# echo never >> /sys/kernel/mm/transparent_hugepage/defrag

[root@prosvr-6cf7-191010 logs]# cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]

[root@prosvr-6cf7-191010 logs]# cat /sys/kernel/mm/transparent_hugepage/defrag
always defer defer+madvise madvise [never]
```

设置机器重启后还生效  
在 /etc/rc.local 中加入如下命令

```text
[root@prosvr-6cf7-191010 logs]# cat <<EOF>> /etc/rc.local
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
    echo never >> /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
    echo never >> /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
```

或者通过修改 grub 启动参数来实现

```text
[root@prosvr-ea03-191010 logs]# cat /etc/default/grub
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet idle=halt biosdevname=0 net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs"
GRUB_DISABLE_RECOVERY="true"
```

将  
`GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet idle=halt biosdevname=0 net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs"`  
这一行修改为  
`GRUB_CMDLINE_LINUX="crashkernel=auto rhgb quiet idle=halt biosdevname=0 net.ifnames=0 console=tty0 console=ttyS0,115200n8 noibrs transparent_hugepage=never"`  
并执行  
`[root@prosvr-ea03-191010 logs]# grub2-mkconfig -o /boot/grub2/grub.cfg`

### 安装 mongodb

* **下载安装包解压**

[https://www.mongodb.com/download-center/community](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.mongodb.com%2Fdownload-center%2Fcommunity)

选择适合自己环境的安装包（此处选择 v4.0.12 centos7 的安装包）

> ![](https://upload-images.jianshu.io/upload_images/9885453-2da14b3347a60cd2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)image.png

三台节点做同样的操作

```text
[root@prosvr-6cf7-191010 packages]# wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.12.tgz
--2019-10-10 21:35:01--  https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.12.tgz
正在解析主机 fastdl.mongodb.org (fastdl.mongodb.org)... 13.249.171.96, 13.249.171.61, 13.249.171.9, ...
正在连接 fastdl.mongodb.org (fastdl.mongodb.org)|13.249.171.96|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：105865380 (101M) [application/x-gzip]
正在保存至: “mongodb-linux-x86_64-rhel70-4.0.12.tgz”
100%[======================================================================================================================================================================>] 105,865,380  755KB/s 用时 47s    
2019-10-10 21:35:50 (2.13 MB/s) - 已保存 “mongodb-linux-x86_64-rhel70-4.0.12.tgz” [105865380/105865380])
```

解压并移动到安装路径下

```text
[root@prosvr-6cf7-191010 packages]# tar -zxvf mongodb-linux-x86_64-rhel70-4.0.12.tgz
mongodb-linux-x86_64-rhel70-4.0.12/THIRD-PARTY-NOTICES.gotools
mongodb-linux-x86_64-rhel70-4.0.12/README
mongodb-linux-x86_64-rhel70-4.0.12/THIRD-PARTY-NOTICES
mongodb-linux-x86_64-rhel70-4.0.12/MPL-2
mongodb-linux-x86_64-rhel70-4.0.12/LICENSE-Community.txt
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongodump
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongorestore
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongoexport
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongoimport
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongostat
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongotop
mongodb-linux-x86_64-rhel70-4.0.12/bin/bsondump
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongofiles
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongoreplay
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongod
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongos
mongodb-linux-x86_64-rhel70-4.0.12/bin/mongo
mongodb-linux-x86_64-rhel70-4.0.12/bin/install_compass

[root@prosvr-6cf7-191010 packages]# ls
docker  mongodb-linux-x86_64-rhel70-4.0.12  mongodb-linux-x86_64-rhel70-4.0.12.tgz
[root@prosvr-6cf7-191010 packages]# mv mongodb-linux-x86_64-rhel70-4.0.12 /usr/local/mongodb
[root@prosvr-6cf7-191010 packages]# cd /usr/local/mongodb/
[root@prosvr-6cf7-191010 mongodb]# ls
bin  LICENSE-Community.txt  MPL-2  README  THIRD-PARTY-NOTICES  THIRD-PARTY-NOTICES.gotools
```

* **设置数据存储路径**

数据库存储路径

```text
[root@prosvr-6cf7-191010 mongodb]# mkdir -p /var/lib/mongodb/data
```

日志路径

```text
[root@prosvr-6cf7-191010 mongodb]# mkdir -p /var/lib/mongodb/logs
```

配置文件（三个节点的配置文件完全一样，除了 bind\_ip，bind\_ip 设置各 db 所在节点的 ip ）

```text
[root@prosvr-6cf7-191010 mongodb]# cat /etc/mongodb.conf
port = 27017
bind_ip = 172.19.30.240
dbpath = /mnt/mongodb/data
logpath = /mnt/mongodb/logs/mongo.log
pidfilepath = /usr/local/mongodb/mongo.pid
fork = true
logappend = true
shardsvr = true
directoryperdb = true
replSet = promongodb
maxConns = 1000000
#auth = true
#keyFile = /usr/local/mongodb/keyfile
```

设置 path 路径

```text
[root@prosvr-6cf7-191010 mongodb]# cat <<EOF>> ~/.bashrc
export PATH="/usr/local/mongodb/bin:$PATH"
EOF

[root@prosvr-6cf7-191010 mongodb]# source ~/.bashrc
```

设置启动脚本 /etc/init.d/mongodb

```text
[root@prosvr-6cf7-191010 mongodb]# chmod 755 /etc/init.d/mongodb

[root@prosvr-6cf7-191010 mongodb]# cat /etc/init.d/mongodb
#!/bin/bash
#
# mongod        Start up the MongoDB server daemon
#
# source function library
. /etc/rc.d/init.d/functions
MONGOD=/usr/local/mongodb/bin/mongod
CONFIG_FILE=/etc/mongodb.conf
start()
{
    $MONGOD -f $CONFIG_FILE
    echo "MongoDB is running background..."
}
stop()
{
    pkill mongod
    echo "MongoDB is stopped."
}
restart()
{
    stop
    sleep 1
    start
}   
case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    *)
        echo $"Usage: $0 {start|stop|restart}"
esac
```

* **依次启动三台 mongodb**

```text
[root@prosvr-6cf7-191010 mongodb]# ln -s /etc/init.d/mongodb /usr/local/bin/mongodb

[root@prosvr-6cf7-191010 mongodb]# mongodb start
```

或

```text
[root@prosvr-6cf7-191010 mongodb]# /usr/local/mongodb/bin/mongod --config /usr/local/mongodb/mongodb.conf

2019-10-10T21:44:29.705+0800 I STORAGE  [main] Max cache overflow file size custom option: 0
about to fork child process, waiting until server is ready for connections.
forked process: 3861
child process started successfully, parent exiting
```

* **三台节点都启动完成后，开始配置 mongodb**

选一台节点进行配置（172.19.20.187）  
执行 mongo 命令登陆

```text
[root@prosvr-6cf7-191010 ~]# mongo 172.19.20.187:27017
MongoDB shell version v4.0.12
connecting to: mongodb://172.19.30.240:27017/test?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("09fc2af9-f216-4b3e-94f0-d8fef6949c1b") }
MongoDB server version: 4.0.12

# 初始化副本集
> rs.initiate()
{
    "info2" : "no configuration specified. Using a default configuration for the set",
    "me" : "172.19.20.187:27017",
    "ok" : 1,
    "operationTime" : Timestamp(1570721887, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1570721887, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
promongodb:PRIMARY> rs.conf()
{
    "_id" : "promongodb",
    "version" : 1,
    "protocolVersion" : NumberLong(1),
    "writeConcernMajorityJournalDefault" : true,
    "members" : [
        {
            "_id" : 0,
            "host" : "172.19.20.187:27017",
            "arbiterOnly" : false,
            "buildIndexes" : true,
            "hidden" : false,
            "priority" : 1,
            "tags" : {
            },
            "slaveDelay" : NumberLong(0),
            "votes" : 1
        }
    ],
    "settings" : {
        "chainingAllowed" : true,
        "heartbeatIntervalMillis" : 2000,
        "heartbeatTimeoutSecs" : 10,
        "electionTimeoutMillis" : 10000,
        "catchUpTimeoutMillis" : -1,
        "catchUpTakeoverDelayMillis" : 30000,
        "getLastErrorModes" : {
        },
        "getLastErrorDefaults" : {
            "w" : 1,
            "wtimeout" : 0
        },
        "replicaSetId" : ObjectId("5d9f505f4e911e5eb22d2b9e")
    }
}
# 添加另外两个复本集从节点
promongodb:PRIMARY> rs.add("172.19.30.240:27017")
{
    "ok" : 1,
    "operationTime" : Timestamp(1570722034, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1570722034, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
promongodb:PRIMARY> rs.add("172.19.40.70:27017")
{
    "ok" : 1,
    "operationTime" : Timestamp(1570722059, 2),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1570722059, 2),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
# 查看节点顺序
promongodb:PRIMARY> cfg = rs.conf()
{
    "_id" : "promongodb",
    "version" : 3,
    "protocolVersion" : NumberLong(1),
    "writeConcernMajorityJournalDefault" : true,
    "members" : [
        {
            "_id" : 0,
            "host" : "172.19.20.187:27017",
            "arbiterOnly" : false,
            "buildIndexes" : true,
            "hidden" : false,
            "priority" : 1,
            "tags" : {
            },
            "slaveDelay" : NumberLong(0),
            "votes" : 1
        },
        {
            "_id" : 1,
            "host" : "172.19.30.240:27017",
            "arbiterOnly" : false,
            "buildIndexes" : true,
            "hidden" : false,
            "priority" : 1,
            "tags" : {
            },
            "slaveDelay" : NumberLong(0),
            "votes" : 1
        },
        {
            "_id" : 2,
            "host" : "172.19.40.70:27017",
            "arbiterOnly" : false,
            "buildIndexes" : true,
            "hidden" : false,
            "priority" : 1,
            "tags" : {
            },
            "slaveDelay" : NumberLong(0),
            "votes" : 1
        }
    ],
    "settings" : {
        "chainingAllowed" : true,
        "heartbeatIntervalMillis" : 2000,
        "heartbeatTimeoutSecs" : 10,
        "electionTimeoutMillis" : 10000,
        "catchUpTimeoutMillis" : -1,
        "catchUpTakeoverDelayMillis" : 30000,
        "getLastErrorModes" : {
        },
        "getLastErrorDefaults" : {
            "w" : 1,
            "wtimeout" : 0
        },
        "replicaSetId" : ObjectId("5d9f505f4e911e5eb22d2b9e")
    }
}
```

* **设置各个节点的优先级**

MongoDB副本集通过设置priority 决定优先级,默认优先级为1,priority值是0到100之间的数字,数字越大优先级越高, `priority=0`,则此节点永远不能成为主节点 primay。

`cfg.members[0].priority =1`参数,中括号里的数字是执行rs.conf\(\)查看到的节点顺序, 第一个节点是0,第二个节点是 1,第三个节点是 2,以此类推。优先级最高的那个被设置为主节点。

```text
promongodb:PRIMARY> cfg.members[0].priority = 1 
1
promongodb:PRIMARY> cfg.members[1].priority = 1
1
promongodb:PRIMARY> cfg.members[2].priority = 2  //设置_ID 为 2 的节点为主节点。即当当前主节点发生故障时，该节点就会转变为主节点接管服务
```

使配置生效

```text
promongodb:PRIMARY> rs.reconfig(cfg)
{
    "ok" : 1,
    "operationTime" : Timestamp(1570722179, 1),
    "$clusterTime" : {
        "clusterTime" : Timestamp(1570722179, 1),
        "signature" : {
            "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
            "keyId" : NumberLong(0)
        }
    }
}
```

* **到另外两台从节点上分别进行配置（读写分离需要）**

```text
[root@prosvr-ea03-191010 ~]# mongo 172.19.30.240:27017
MongoDB shell version v4.0.12
connecting to: mongodb://172.19.30.240:27017/test?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("f7e1269d-095e-404e-addd-9c9fd4fa1d2a") }
MongoDB server version: 4.0.12

promongodb:SECONDARY> db.getMongo().setSlaveOk() //设置从节点为只读.注意从节点的前缀现在是SECONDARY。看清楚才设置
```

```text
[root@prosvr-ea03-191010 ~]# mongo 172.19.40.70:27017
MongoDB shell version v4.0.12
connecting to: mongodb://172.19.40.70:27017/test?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("f7e1269d-095e-404e-addd-9c9fd4fa1d2a") }
MongoDB server version: 4.0.12
promongodb:SECONDARY> db.getMongo().setSlaveOk() //设置从节点为只读.注意从节点的前缀现在是SECONDARY。看清楚才设置
```

设置完只读操作后，从节点将不能执行相关操作

```text
promongodb:SECONDARY> show dbs
2019-10-12T11:22:42.646+0800 E QUERY    [js] Error: listDatabases failed:{
    "ok" : 0,
    "errmsg" : "not master and slaveOk=false",
    "code" : 13435,
    "codeName" : "NotMasterNoSlaveOk"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:139:1
shellHelper.show@src/mongo/shell/utils.js:882:13
shellHelper@src/mongo/shell/utils.js:766:15
@(shellhelp2):1:1
```

* **设置数据库账号,开启登录验证**

```text
# 创建账号

[root@prosvr-6cf7-191010 ~]# mongo 172.19.20.187:27017
MongoDB shell version v4.0.12
connecting to: mongodb://172.19.20.187:27017/?gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("4b65920c-f7e7-42af-ae43-3e9ab3bbba89") }
MongoDB server version: 4.0.12
Welcome to the MongoDB shell.

promongodb:PRIMARY> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
promongodb:PRIMARY> use admin
switched to db admin
promongodb:PRIMARY> show collections
system.version
promongodb:PRIMARY> db.createUser({user:"wood",pwd:”xxxxxxx",roles:[{role:"root",db:"admin"}]})
Successfully added user: {
    "user" : "wood",
    "roles" : [
        {
            "role" : "root",
            "db" : "admin"
        }
    ]
}
promongodb:PRIMARY> db.auth("wood","xxxxxxx")
1
promongodb:PRIMARY> db.createUser({user:'admin', pwd:’aaxxxxxxx', roles:[{ role:"userAdminAnyDatabase",db:"admin"}]})
Successfully added user: {
    "user" : "admin",
    "roles" : [
        {
            "role" : "userAdminAnyDatabase",
            "db" : "admin"
        }
    ]
}
promongodb:PRIMARY> db.auth("admin”,"aaxxxxxxx")
1
promongodb:PRIMARY> show collections
system.users
system.version
promongodb:PRIMARY> db.system.users.find(
... )
{ "_id" : "admin.wood", "userId" : UUID("97a7bd6axxxxxxx92-0380abc45d97"), "user" : "wood", "db" : "admin", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "xxxxxxx+UVZLSCQ==", "storedKey" : “xxxxxxxx=", "serverKey" : “xxxxxxxx=" } }, "roles" : [ { "role" : "root", "db" : "admin" } ] }
{ "_id" : "admin.admin", "userId" : UUID("5d566ac1xxxxxxx0f3-0445ba50e2cb"), "user" : "admin", "db" : "admin", "credentials" : { "SCRAM-SHA-1" : { "iterationCount" : 10000, "salt" : "gDxxxxxxxYwaA==", "storedKey" : "/vcccxxxxxxx=", "serverKey" : "+xxxxxxxSmaJXBo=" } }, "roles" : [ { "role" : "userAdminAnyDatabase", "db" : "admin" } ] }
promongodb:PRIMARY>
```

* **设置 keyfile，用于集群节点之间的通信**

副本集服务器,开启`--auth`参数的同时,必须指定 keyfile 参数,节点之间的通讯基于该 keyfile,key 长度必须在 6 到 1024 个字符之间,

最好为 3 的倍数,不能含有非法字符。

```text
[root@prosvr-dd0e-191010 ~]# cd /usr/local/mongodb/
[root@prosvr-dd0e-191010 mongodb]# openssl rand -base64 21 > keyfile  
[root@prosvr-dd0e-191010 mongodb]# ls
bin  keyfile  LICENSE-Community.txt  mongo.pid  MPL-2  README  THIRD-PARTY-NOTICES  THIRD-PARTY-NOTICES.gotools
[root@prosvr-dd0e-191010 mongodb]# cat keyfile
tBZnTjxxxxxxxye/kxIb21JYnJlZ
[root@prosvr-dd0e-191010 mongodb]#  chmod 600 /usr/local/mongodb/keyfile
```

去掉 auth=true keyfile= /topath/kefile 的注释

```text
[root@prosvr-dd0e-191010 mongodb]# cat /etc/mongodb.conf
port = 27017
bind_ip = 172.19.40.70
dbpath = /mnt/mongodb/data
logpath = /mnt/mongodb/logs/mongo.log
pidfilepath = /usr/local/mongodb/mongo.pid
fork = true
logappend = true
shardsvr = true
directoryperdb = true
replSet = promongodb
maxConns = 1000000
auth = true
keyFile = /usr/local/mongodb/keyfile
```

* **重启 mongodb 服务**

```text
[root@prosvr-dd0e-191010 mongodb]# mongodb stop
[root@prosvr-dd0e-191010 mongodb]# mongodb start
```

将生成的 keyfile 复制到其他两个节点上，并修改同样的配置，重启服务使其生效

```text
[root@prosvr-dd0e-191010 mongodb]# cat keyfile
tBZnTjxxxxxxxye/kxIb21JYnJlZ
```

* **依次设置三台 mongodb 的 logrotate**

```text
[root@prosvr-dd0e-191010 logs]# cat /etc/logrotate.d/mongodb

/mnt/mongodb/logs/*.log {
    rotate 4
    maxsize 128M
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d-%s
    compress
}
```

PS: 若登陆 mongo 时，有 warning 提示

```text
2019-10-10T16:10:02.004+0800 I CONTROL  [initandlisten] ** WARNING: soft rlimits too low. rlimits set to 31813 processes, 65535 files. Number of processes should be at least 32767.5 : 0.5 times number of files.
```

则可通过设置 mac process limit 超过提示项，并重启 mongodb 即可

一种方式  
通过获取 mongodb 的 pid 查看详细情况

```text
[root@prosvr-ea03-191010 ~]# ps -efl |grep mongodb
1 S root      1667     1  0  80   0 - 362746 -     12:21 ?        00:00:23 /usr/local/mongodb/bin/mongod -f /etc/mongodb.conf
0 S root      1931  1560  0  80   0 - 28184 -      13:46 pts/0    00:00:00 grep --color=auto mongodb
[root@prosvr-ea03-191010 ~]# cat /proc/1667/limits
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
**Max processes             31813                31813                processes**
Max open files            65535                65535                files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       31813                31813                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us  
```

```text
[root@prosvr-ea03-191010 logs]# ulimit -u # 发现系统的 user max processes 即为 31813
31813
# 修改  user max processes 为 42000
[root@prosvr-ea03-191010 logs]# ulimit -u 42000
# 并将改命令写入 /etc/profile
[root@prosvr-ea03-191010 logs]# echo "ulimit -u 42000” >> /etc/profile
# 重启 mongodb 后，再次查看 mongodb 进程的使用情况
[root@prosvr-ea03-191010 ~]# cat /proc/`pgrep mongod`/limits
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
**Max processes             42000                42000                processes**
Max open files            65535                65535                files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       31813                31813                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us
```

再次 mongo 登陆服务则不再提示

* **开启 mongodb 自身的监控服务**

```text
[https://cloud.mongodb.com/freemonitoring/cluster/VJPV34K25T6P4EIPLVBWTFXEA2W2TUMP](https://cloud.mongodb.com/freemonitoring/cluster/VJPV34K25T6P4EIPLVBWTFXEA2W2TUMP) 
promongodb:PRIMARY> db.enableFreeMonitoring()
{
    "state" : "enabled",
    "message" : "To see your monitoring data, navigate to the unique URL below. Anyone you share the URL with will also be able to view this page. You can disable monitoring at any time by running db.disableFreeMonitoring().",
    "url" : "https://cloud.mongodb.com/freemonitoring/cluster/VXXXXXXXXXXXUMP",
    "userReminder" : "",
    "ok" : 1
}
```

关闭监控服务

```text
promongodb:PRIMARY> db.disableFreeMonitoring()

![image.png](https://upload-images.jianshu.io/upload_images/9885453-db4583a055b68fc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
```

### 客户端连接使用副本集

> MongoDB复制集里Primary节点是不固定的，当遇到复制集轮转升级、Primary宕机、网络分区等场景时，复制集可能会选举出一个新的Primary，而原来的Primary则会降级为Secondary，

> 即发生主备切换。当连接复制集时，如果直接指定Primary的地址来连接，当时可能可以正确读写数据的，但一旦复制集发生主备切换，你连接的Primary会降级为Secondary，将无法继续执行写操作，这将严重影响到你的线上服务。

> 要正确连接复制集，需要先了解下MongoDB的[Connection String URI](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.mongodb.com%2Fmanual%2Freference%2Fconnection-string%2F)，所有官方的driver都支持以Connection String的方式来连接MongoDB。

下面就是Connection String包含的主要内容

> mongodb://\[[username:password@\]host1\[:port1\]\[,host2\[:port2\],...\[,hostN\[:portN\]\]\]\[/\[database\]\[?options\]\]](https://links.jianshu.com/go?to=username%3Apassword%40%255Dhost1%255B%3Aport1%255D%255B%2Chost2%255B%3Aport2%255D%2C...%255B%2ChostN%255B%3AportN%255D%255D%255D%255B%2F%255Bdatabase%255D%255B%3Foptions%255D%255D)
>
> * mongodb:// 前缀，代表这是一个Connection String
> * username:password@ 如果启用了鉴权，需要指定用户密码
> * hostX:portX 复制集成员的ip:port信息，多个成员以逗号分割
> * /database 鉴权时，用户帐号所属的数据库
> * ?options 指定额外的连接选项

* **例如 java 程序**

```text
MongoClientURI connectionString = new MongoClientURI("mongodb://wood:xxxxx@172.19.30.240:27017,172.19.20.187:27017,172.19.40.70:27017/admin?replicaSet=promongodb"); 
MongoClient client = new MongoClient(connectionString);
MongoDatabase database = client.getDatabase(“yourDatabase");
MongoCollection collection = database.getCollection(“yourCollections");
```

* **实现读写分离**

主节点负责所有的读写操作将会造成主节点压力较大，通过客户端程序实现读写分离

> 读参数一共有六个参数：primary、primaryPreferred、secondary、secondary、secondaryPreferred、nearest。
>
> * primary：默认参数，只从主节点上进行读取操作；
> * primaryPreferred：大部分从主节点上读取数据,只有主节点不可用时从secondary节点读取数据。
> * secondary：只从secondary节点上进行读取操作，存在的问题是secondary节点的数据会比primary节点数据“旧”。
> * secondaryPreferred：优先从secondary节点进行读取操作，secondary节点不可用时从主节点读取数据；
> * nearest：不管是主节点、secondary节点，从网络延迟最低的节点上读取数据。

读写分离做好后，就可以进行数据分流，减轻压力，解决了"主节点的读写压力过大如何解决？"这个问题。

不过当副本节点增多时，主节点的复制压力会加大有什么办法解决吗？基于这个问题，Mongodb已有了相应的解决方案 - 引用仲裁节点：

在Mongodb副本集中，仲裁节点不存储数据，只是负责故障转移的群体投票，这样就少了数据复制的压力。

> 其实不只是主节点、副本节点、仲裁节点，还有Secondary-Only、Hidden、Delayed、Non-&gt; Voting，其中：
>
> * Secondary-Only：不能成为primary节点，只能作为secondary副本节点，防止一些性能不高的节点成为主节点。
> * Hidden：这类节点是不能够被客户端制定IP引用，也不能被设置为主节点，但是可以投票，一般用于备份数据。
> * Delayed：可以指定一个时间延迟从primary节点同步数据。主要用于备份数据，如果实时同步，误删除数据马上同步到从节点，恢复又恢复不了。
> * Non-Voting：没有选举权的secondary节点，纯粹的备份数据节点。

java 程序为例。ReadPreference preference = ReadPreference.secondary\(\); 设置

```text
public class TestMongoDBReplSet {
    public static void main(String[] args)  {
        try {
            List<ServerAddress> addresses = new ArrayList<ServerAddress>();  
            ServerAddress address1 = new ServerAddress("172.19.40.70" , 27017);
            ServerAddress address2 = new ServerAddress("172.19.20.187" , 27017);
            ServerAddress address3 = new ServerAddress("172.19.30.240" , 27017);
            addresses.add(address1);  
            addresses.add(address2);
            addresses.add(address3);
            MongoClient client = new MongoClient(addresses);
            DB db = client.getDB( "test");
            DBCollection coll = db.getCollection( "test");
            BasicDBObject object = new BasicDBObject();  
            object.append( "key1", "value1" );
           ** ReadPreference preference = ReadPreference.secondary();**  
            DBObject dbObject = coll.findOne(object, null , preference);  
            System. out .println(dbObject);  
        } catch (Exception e) {
            e.printStackTrace();  
        }
    }
}
```

