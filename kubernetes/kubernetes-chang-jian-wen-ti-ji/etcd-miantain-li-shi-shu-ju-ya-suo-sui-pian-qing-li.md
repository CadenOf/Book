# etcd miantain 历史数据压缩碎片清理

### overview

由于etcd保留了集群中所有版本的确切的历史记录，所以需要定期压缩历史记录以避免性能下降及最终将存储空间耗尽。etcd集群需要定期维护才能保持可靠性，etcd的维护通常可以自动执行，无需停机，也不会影响性能下降。

压缩历史数据会丢弃给定的压缩版本之前所有的历史数据，在压缩数据后，后端数据存储可能会出现内部碎片，碎片仍会占用存储空间，因此需要对碎片进行整理将空间释放回文件系统。

### etcd 维护流程

* **备份etcd数据**

```text
$ cat etcd-snapshot  #etcd-snapshot脚本
#!/bin/bash
BACKUP_PATH=/var/lib/etcd-backup
mkdir -p $BACKUP_PATH
SNAP_FILE="etcd-snapshot-`date +%Y%m%d-%H%M%S`"
. /etc/etcd/etcdrc; /usr/local/bin/etcdctl snapshot save "${BACKUP_PATH}/${SNAP_FILE}"
cd $BACKUP_PATH; tar -czvf ${SNAP_FILE}.tar.gz $SNAP_FILE
cd $BACKUP_PATH; find ./ -mtime +7 -type f -delete
rm -f ${BACKUP_PATH}/${SNAP_FILE}
```

```text
$ sh etcd-snapshot #执行脚本备份

2019-01-22 18:28:41.665959 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
Snapshot saved at /var/lib/etcd-backup/etcd-snapshot-20190122-182841
etcd-snapshot-20190122-182841
```

* **获取当前etcd集群所有member endpoints**

  * 导入ENV

  ```text
  $ cat etcdrc 
  export ETCDCTL_API=3
  export ETCDCTL_ENDPOINTS="127.0.0.1:2379"
  export ETCDCTL_CACERT=/etc/etcd/ssl/ca.crt
  export ETCDCTL_CERT=/etc/etcd/ssl/etcd.crt
  export ETCDCTL_KEY=/etc/etcd/ssl/etcd.key
  $ source etcdrc # 执行
  ```

  * 获取member endpoints

  ```text
  $ etcdctl member list
  b647a4c5532e517, started, vms70322, https://10.5.xx.93:2380, https://10.5.xx.93:2379
  c7eeae8975872e8, started, vms70324, https://10.5.xx.95:2380, https://10.5.xx.95:2379
  29a57fa29bbcf1a7, started, vms70323, https://10.5.xx.94:2380, https://10.5.xx.94:2379
  caedcb6700c3f637, started, vms70321, https://10.5.xx.92:2380, https://10.5.xx.92:2379
  f5c43730caf9ca87, started, vms70320, https://10.5.xx.91:2380, https://10.5.xx.91:2379
  ```

* **获取当前最新的etcd revision版本$rev\_num**

```text
$ etcdctl endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*'
39042173

```

* **保留最新的100个revision版本，压缩\($rev\_num -100\)旧版本数据**

```text
$ etcdctl --command-timeout=300s compact 39042073
```

* **碎片整理（对每个member逐一进行，先非leader，再leader）**

```text
$ etcdctl --command-timeout=300s defrag  --endpoints=https://10.5.xx.xx:2379
Finished defragmenting etcd member[https://10.xx.xx.xxx:2379]
```

* **维护结果**

  数据压缩完成之后数据大小DB size将降低：

![](../../.gitbook/assets/image%20%281%29.png)

![](../../.gitbook/assets/image%20%285%29.png)

