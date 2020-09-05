# linux inode 索引节点爆满问题

### **【 问题描述 】**

机器磁盘 `df -h` 查看磁盘存储空间还剩很多，但某些写入但命令却执行不了，通常会报 `no space left on device` ，例如常见的 `npm` 编译时

```text
npm WARN tar ENOSPC: no space left on device, open /node_modules/.staging
```

### **【 问题原因 】**

Linux 的文件系统存储文件有两部份，**Block & Inode**

* **Block** 是用来存储数据用的，也就是我们执行 `df -h` 显示的数据存储情况。

```text
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        3.9G     0  3.9G   0% /dev
tmpfs           3.9G     0  3.9G   0% /dev/shm
tmpfs           3.9G  512K  3.9G   1% /run
tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
/dev/vda1        59G   29G   28G  52% /
overlay          59G   29G   28G  52% /var/lib/docker/overlay2/8af0d652478354ecd03314d3e54d0798473266f73a731f1d5884b17ec1f0505f/merged
shm              64M     0   64M   0% /var/lib/docker/containers/a601bce7eadd08aebe343dc9ecfee55b14b8f3082c4e2e7e5139a9c9178bae75/mounts/shm
tmpfs           784M     0  784M   0% /run/user/0
```

* **Inode** 用来存储这些数据的属性信息，这些信息包括文件大小、用户、用户组、读写权限等。inode 为每个文件进行信息索引，所以就有了 inode的 数值。操作系统根据指令，能通过 inode 值最快的找到相对应的文件。 可通过执行 `df -ih` 来查看 inode 的存储情况，可见在根目录 `\` 存储未满的情况下，inode 存储已经到了 100% 

```text
[root@fat-company-website-200810 ~]# df -ih
Filesystem     Inodes IUsed IFree IUse% Mounted on
devtmpfs         977K   328  977K    1% /dev
tmpfs            980K     1  980K    1% /dev/shm
tmpfs            980K   424  979K    1% /run
tmpfs            980K    16  980K    1% /sys/fs/cgroup
/dev/vda1        3.8M  3.8M  0     100% /
overlay          3.8M  3.2M  593K   85% /var/lib/docker/overlay2/8af0d652478354ecd03314d3e54d0798473266f73a731f1d5884b17ec1f0505f/merged
shm              980K     1  980K    1% /var/lib/docker/containers/a601bce7eadd08aebe343dc9ecfee55b14b8f3082c4e2e7e5139a9c9178bae75/mounts/shm
tmpfs            980K     1  980K    1% /run/user/0
```

### **【 解决办法 】**

{% hint style="success" %}
**临时解决方案**：找出占用 inode 存储异常大的文件存储，看是否可删除将其删掉可临时恢复。
{% endhint %}

  执行命令`for i in /*; do echo $i; find $i | wc -l; done` 从 `\` 开始，例举出各个目录下的文件数（文件数较多的情况下，该命令会卡在文件数多的文件目录下，`ctl + c` 再继续查找该目录即可）

```text
$ for i in /*; do echo $i; find $i | wc -l; done
/bin
1
/boot
325
/data
40114
/dev
330
/etc
2485
/home
1
/proc
43800
...
......
27968
/tmp
9
/usr
149732
/var
3310833
```

  可见在 `usr` `var` 两个目录下的文件数最多，再依次查看

```text
$ for i in /var/*; do echo $i; find $i | wc -l; done
/var/adm
1
...
....

/var/lib
3310365
/var/local
3

$ for i in /var/lib/*; do echo $i; find $i | wc -l; done
/var/lib/dbus
1
/var/lib/dhclient
2
/var/lib/docker
3303687

$ for i in /var/lib/docker/*; do echo $i; find $i | wc -l; done
/var/lib/docker/network
3
/var/lib/docker/overlay2
3302187

```

  至此可知该场景下是 docker 容器中遗留的文件存储问题，有太多的编译产生的缓存文件，简单的处理方法为直接清理 docker 系统

```text
$ docker system prune
WARNING! This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all dangling images
        - all dangling build cache
Are you sure you want to continue? [y/N] y
Deleted Containers:

77651c2019fbe89e6b2e89e9803ac5658a341c6ae29e3e885ae1f9170a6269b8
fa4f19b88c377ce49f257af750e61a81c7cc43729291be93bc0d2c4b87bf5276
c86ca6d464441491c949c8d358cb980f6c4990a594b520b73f961ead5d7e25c7
3d8931a1adddb6c4ed465a28f8d16422d1ba04a70d3777276c49c790cce8f810

Deleted Images:
deleted: sha256:2a512322a0b472e9f013326937fc8ede015d19199ae95e0764805e3accdc8e3e

.....
Total reclaimed space: 7.763GB
```

  再查看 inode 存储情况时, `\` 的 inode 存储已降到了 11%

```text
$  df -ih
Filesystem     Inodes IUsed IFree IUse% Mounted on
devtmpfs         977K   328  977K    1% /dev
tmpfs            980K     1  980K    1% /dev/shm
tmpfs            980K   424  979K    1% /run
tmpfs            980K    16  980K    1% /sys/fs/cgroup
/dev/vda1        3.8M  393K  3.4M   11% /
overlay          3.8M  393K  3.4M   11% /var/lib/docker/overlay2/8af0d652478354ecd03314d3e54d0798473266f73a731f1d5884b17ec1f0505f/merged
shm              980K     1  980K    1% /var/lib/docker/containers/a601bce7eadd08aebe343dc9ecfee55b14b8f3082c4e2e7e5139a9c9178bae75/mounts/shm
tmpfs            980K     1  980K    1% /run/user/0
```

{% hint style="success" %}
**长远解决方案**： case by case！
{% endhint %}

* 如果 case 为上述 `npm` 编译容器没有清理的问题，可在编译出 dist 后自动清理 node\_modules
* 如果是 crontab 里面定时执行的句子里没有加 `> /dev/null 2>&1` , 系统中 cron 执行的程序有输出内容，输出内容会以邮件形式发给 cron 的用户，而 `sendmail` 没有启动所以就产生了很大零碎的文件.。

