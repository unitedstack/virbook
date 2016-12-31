# 检查点快照

检查点快照保存了虚拟机的磁盘和内存信息。libvirt提供了`virDomainSnapshotCreateXML()`方法来创建检查点快照，`virDomainRevertToSnapshot()`方法从检查点快照恢复。

可通过`virsh snapshot-create-as/snapshot-revert`来管理检查点快照。

##创建快照

检查点快照的创建比较简单，需要同时指定"--memspec"和"--diskspec"参数，使用方式如下：
```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-create-as instance-0000005a --memspec file=/root/05a.xml,snapshot=external --diskspec vda,snapshot=external
Domain snapshot 1483089005 created
```

通过virsh命令查看快照信息，如下：
```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-list instance-0000005a
 Name                 Creation Time             State
------------------------------------------------------------
 1483080871           2016-12-30 14:54:31 +0800 running
 1483086322           2016-12-30 16:25:22 +0800 disk-snapshot
 1483087069           2016-12-30 16:37:49 +0800 disk-snapshot
 1483089005           2016-12-30 17:10:05 +0800 running
```

## 快照恢复

上面在创建检查点快照时，磁盘快照的类型为external,在磁盘快照章节已提到，目前不支持磁盘外部快照的恢复。这里也不支持检查点的恢复。
```bash
[root@compute1 e03ce663-3ca8-4afd-8199-77052911cfa5(keystone_admin)]# virsh snapshot-revert instance-0000005a 1483089005
error: unsupported configuration: revert to external snapshot not supported yet
```