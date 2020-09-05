# Linux umount target is busy 挂盘卸载问题

### 【 **问题描述 】**

Linux 下挂载后的分区或者磁盘某些时候需要 umount 的时候出现

```text
root@ip-10-19-2-231:~# umount /mnt
umount: /mnt: target is busy.
```

### **【 问题原因 】**

该报错通常是由于待卸载磁盘正在被使用，导致无法直接卸载。需要将当前使用数据盘的进程杀掉，才能卸载。

### **【 解决办法 】**

#### \*\*\*\*🔨 ****使用 fuser 工具

安装 fuser 工具

```text
root@ip-10-19-2-231:~# yum install psmisc -y
```

查看在使用的进程

```text
root@ip-10-19-2-231:~# fuser -mv /mnt/
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /mnt
                     root       3921 ..c.. bash
                     root       4044 ..c.. tmux: server
                     root       4045 ..c.. bash
                     root       5365 ..c.. bash
                     root       5508 ..c.. bash
                     root      21436 ..c.. bash
                     root      22528 ..c.. bash
                     root      23290 ..c.. bash
                     root      30856 ..c.. bash
```

杀死占用的进程

```text
root@ip-10-19-2-231:~# fuser -kv /mnt/
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /mnt
                     root       5508 ..c.. bash
                     
root@ip-10-19-2-231:~# fuser -mv /mnt/
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /mnt
```

> fuser 参数说明：  
> -k,    --kill            kill processes accessing the named file  
> -m,  --mount     show all processes using the named filesystems or block device  
> -v,    --verbose   verbose output

> 注意：  
> 可以使用 fuser -km /mnt 进行 kill 进程。  
> 可以使用 kill 命令杀掉查到对应的进程 。  
> 强制 kill 进程可能会导致数据丢失，请确保数据得到有效备份后，再进行相关操作。

确认无进程连接后，使用卸载命令

```text
[root@server-10 ~]# umount /mnt/
[root@server-10 ~]# 
```

{% hint style="info" %}
若上述方式不能解决问题，可尝试下述方式
{% endhint %}

\*\*\*\*🔨 使用 **lsof 工具处理**

```text
root@ip-10-19-2-231:~# lsof /mnt/
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
tmux:\x20  4044 root  cwd    DIR   0,51        0   25 /mnt/users
bash       4045 root  cwd    DIR   0,51        0   25 /mnt/users
bash       5365 root  cwd    DIR   0,51        0 6550 /mnt/Bosch
bash       5508 root  cwd    DIR   0,51        0    1 /mnt
bash      21436 root  cwd    DIR   0,51        0 1418 /mnt/users/c/ffhq-dataset
bash      22528 root  cwd    DIR   0,51        0 6559 /mnt/BDD100K
bash      23290 root  cwd    DIR   0,51        0 6573 /mnt/CURE-TSD
bash      30856 root  cwd    DIR   0,51        0 3289 /mnt/users/x
```

找到 PID 对应的进程或者服务，然后杀死或者停止相应服务，例如

```text
root@ip-10-19-2-231:~# kill -9 4044
```

再次查看

```text
root@ip-10-19-2-231:~# lsof /mnt/
root@ip-10-19-2-231:~# fuser -mv /mnt/
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /mnt
```

现在可以直接卸载了

```text
root@ip-10-19-2-231:~# umount /mnt
```

> 参考：[https://www.cnblogs.com/ding2016/p/9605526.html](https://www.cnblogs.com/ding2016/p/9605526.html)



