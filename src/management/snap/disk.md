# 磁盘快照

磁盘快照都由一个qcow2中的snapshot table来维护。起初该table是空的，当打完一个磁盘快照后，包含L1 table的一个复制，当L2 table或者data cluster改变的时候，则会将原来的数据复制一份，由snapshot的L1 table来维护，而原来的data cluster已经改变，在原地。


磁盘快照的创建和恢复分别调用的是libvirt的`virDomainSnapshotCreateXML(flags = VIR_DOMAIN_SNAPSHOT_CREATE_DISK_ONLY)`和`virDomainRevertToSnapshot`两个API。可通过`virsh snapshot-create/snapshot-create-as/snapshot-revert`来管理磁盘快照。

## 创建快照
### 创建内部快照

下面我们通过virsh snapshot-create来创建快照。创建快照时虚机处于paused状态，快照完成后变为running状态。持续时间较长。

具体操作如下：
```bash
[root@compute1 ~(keystone_admin)]# virsh list
 Id    Name                           State
----------------------------------------------------
 30    instance-0000005a              running
 31    instance-0000005b              running
 32    instance-0000005c              running
 33    instance-0000005d              running

创建快照
[root@compute1 ~(keystone_admin)]# virsh snapshot-create instance-0000005a
Domain snapshot 1483080871 created
```
可以看到快照创建成功。下面通过virsh工具查看该虚拟机的快照信息：
```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-list instance-0000005a
 Name                 Creation Time             State
------------------------------------------------------------
 1483080871           2016-12-30 14:54:31 +0800 running
```

该快照信息也可以通过qemu-img工具在虚拟机的磁盘文件中查看：
```bash
[root@compute1 2a6b2314-08d0-4b14-a945-f12dffd42bb9(keystone_admin)]# qemu-img info disk
image: disk
file format: qcow2
virtual size: 1.0G (1073741824 bytes)
disk size: 53M
cluster_size: 65536
backing file: /var/lib/nova/instances/_base/b325fa40c745c85e39416655521674088dec542e
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         1483080871              51M 2016-12-30 14:54:31  164:54:15.811
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```
下面我们来查看快照的xml文件：

```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-dumpxml instance-0000005a 1483080871
<domainsnapshot>
  <name>1483080871</name>
  <state>running</state>
  <creationTime>1483080871</creationTime>
  <memory snapshot='internal'/>
  <disks>
    <disk name='vda' snapshot='internal'/>     # 内部快照
  </disks>
  ...
  <domain type='qemu'>
    <devices>
      <emulator>/usr/libexec/qemu-kvm</emulator>
      <disk type='file' device='disk'>
        <driver name='qemu' type='qcow2' cache='none'/>
        <source file='/var/lib/nova/instances/2a6b2314-08d0-4b14-a945-f12dffd42bb9
        <target dev='vda' bus='virtio'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/
      </disk>
      ...
  </domain>
</domainsnapshot>

```

### 创建外部快照

virsh snapshot-create不支持创建外部快照，我们可以使用virsh snapshot-create-as来指定"--disk-only"和"--diskspec"参数来给磁盘创建外部快照。"--disk-only"参数会对虚拟机的所有磁盘做外部快照，不包含内存的快照。

还是以上述做测试的虚拟机为例做快照相关操作。

```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-create-as instance-0000005a --diskspec vda,snapshot=external --disk-only
Domain snapshot 1483086322 created
```

virsh查看虚拟机的快照信息：

```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-list instance-0000005a
 Name                 Creation Time             State
------------------------------------------------------------
 1483080871           2016-12-30 14:54:31 +0800 running
 1483086322           2016-12-30 16:25:22 +0800 disk-snapshot
```
此时快照的xml文件部分如下：

```
<domainsnapshot>
  <name>1483086322</name>
  <state>disk-snapshot</state>    # 快照状态
  <parent>                        # 父快照信息
    <name>1483080871</name>
  </parent>
  <creationTime>1483086322</creationTime>
  <memory snapshot='no'/>
  <disks>
    <disk name='vda' snapshot='external' type='file'>    # 外部快照
      <driver type='qcow2'/>
      <source file='/var/lib/nova/instances/2a6b2314-08d0-4b14-a945-f12dffd42bb9/disk.1483086322'/>
    </disk>
  </disks>
  ...
  <domain type='qemu'>
    <devices>
      <emulator>/usr/libexec/qemu-kvm</emulator>
      <disk type='file' device='disk'>
        <driver name='qemu' type='qcow2' cache='none'/>
        <source file='/var/lib/nova/instances/2a6b2314-08d0-4b14-a945-f12dffd42bb9
        <target dev='vda' bus='virtio'/>
        <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/
      </disk>
      ...
  </domain>
</domainsnapshot>
```






## 快照恢复

内部快照的恢复比较简单。我们可以通过以下命令恢复快照文件：

```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-revert instance-0000005a 1483080871 
```
但是目前还不支持回滚到某一个外部磁盘快照。
```bash
[root@compute1 ~(keystone_admin)]# virsh snapshot-revert instance-0000005a 1483086322
error: unsupported configuration: revert to external snapshot not supported yet
```

## 参考文档

1. Qemu Libvirt & KVM：http://leavot.chaoxiyujia.com/wp-content/uploads/2015/07/LibvirtQemuKVM%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5%E8%AE%B2%E7%9A%84%E5%BE%88%E6%B8%85%E6%99%B0.pdf
