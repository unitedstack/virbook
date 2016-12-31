# 内存快照

内存快照只保存虚拟机当前的内存状态。

libvirt提供了`virDomainSave()`,`virDomainSaveFlags`,and `virDomainManagedSave()`三个方法来创建内存快照，提供了`virDomainRestore()`, `virDomainRestoreFlags()`, `virDomainCreate()`, and `virDomainCreateWithFlags()`来做快照的恢复。

可通过`virsh save/restore`来管理内存快照。通过virsh save创建快照时，虚拟机会被关机。

## 创建快照

创建内存快照的操作如下：
```bash
[root@compute1 2a6b2314-08d0-4b14-a945-f12dffd42bb9(keystone_admin)]# virsh save instance-0000005a
Domain instance-0000005a saved to 05a
```
通过这种方式创建快照时，信息被保存在自己添加的文件中，通过virsh snapshot-list无法查看到该快照信息。

## 快照恢复

快照恢复的操作如下：
```bash
[root@compute1 2a6b2314-08d0-4b14-a945-f12dffd42bb9(keystone_admin)]# virsh restore 05a
Domain restored from 05a
```