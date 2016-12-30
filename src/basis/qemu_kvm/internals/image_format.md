# 镜像格式

本章节主要讲述raw，qcow2，vmdk，vdi四种镜像的格式。



## 术语
### sparse文件
sparse文件是Linux文件系统的一个高级特性，能够实现磁盘的超负载使用（overload）。简单举例来说，当用户申请了一块20G的存储空间，但是没有向该空间中写入数据，此时文件系统为了节省存储资源，提高资源利用率，不会分配实际存储空间，只有当真正写入数据时，操作系统才真正一点一点地分配空间，比如一次64KB。于是这个文件看起来很大，而占用空间很小，实际占用空间只与用户填的数据量有关。
## 格式说明

###1. raw

raw格式的最大特点是简单，使用`dd`命令就可以模拟一个raw格式的镜像。且容易转换为其他的格式。支持sparse文件。

```bash
[root@compute1 ~(keystone_admin)]# dd if=/dev/zero of=zero.raw bs=1024k count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB) copied, 16.7809 s, 256 MB/s
```
下面我们使用qemu-img创建4G的raw文件：
```bash
[root@compute1 ~(keystone_admin)]# qemu-img create -f raw test.raw 4G
Formatting 'test.raw', fmt=raw size=4294967296
[root@compute1 ~(keystone_admin)]# qemu-img info test.raw 
image: test.raw
file format: raw
virtual size: 4.0G (4294967296 bytes)
disk size: 0
```
其中virtual size为预分配空间，disk size为实际占用空间。从空间使用上来看，raw格式类似磁盘，使用多少就是多少。

raw格式还可以随时在原来的盘上追加空间，操作如下：
```bash
[root@compute1 ~(keystone_admin)]# qemu-img info test.raw 
image: test.raw
file format: raw
virtual size: 4.0G (4294967296 bytes)
disk size: 0
[root@compute1 ~(keystone_admin)]# qemu-img info zero.raw 
image: zero.raw
file format: raw
virtual size: 4.0G (4294967296 bytes)
disk size: 4.0G
[root@compute1 ~(keystone_admin)]# cat zero.raw test.raw > new.raw
[root@compute1 ~(keystone_admin)]# qemu-img info new.raw 
image: new.raw
file format: raw
virtual size: 8.0G (8589934592 bytes)
disk size: 8.0G
```
###2. qcow2

**qcow2（QEMU copy on write）的基本原理**
qcow2 镜像格式是 QEMU 模拟器支持的一种磁盘镜像。它也是可以用一个文件的形式来表示一块固定大小的块设备磁盘。与普通的 raw 格式的镜像相比，有以下特性：

- 更小的空间占用，即使文件系统不支持空洞(holes)；
- 支持写时拷贝（COW, copy-on-write），镜像文件只反映底层磁盘的变化；
- 支持快照（snapshot），镜像文件能够包含多个快照的历史；
- 可选择基于 zlib 的压缩方式
- 可以选择 AES 加密

原理的具体实现方式可参考：https://www.ibm.com/developerworks/cn/linux/1409_qiaoly_qemuimgages/

QEMU/KVM虚拟机的磁盘内部快照只支持qcow2格式。 
在OpenStack环境中，使用ceph做后端存储时，为了实现Nova、Glance、Cinder统一存储快速创建虚拟机，必须使用raw格式的镜像，如果使用qcow2会导致创建虚拟机过程很慢（Nova会先下载镜像后台转成raw格式再启动）。

和raw格式相比，qcow2格式不支持sparse文件。
```bash
[root@compute1 ~(keystone_admin)]# qemu-img info test.raw 
image: test.raw
file format: raw
virtual size: 4.0G (4294967296 bytes)
disk size: 0
[root@compute1 ~(keystone_admin)]# qemu-img info test.qcow2 
image: test.qcow2
file format: qcow2
virtual size: 4.0G (4294967296 bytes)
disk size: 196K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```
可以看到，同样是4G的文件，test.raw不占用磁盘空间，test.qcow2占用了196k磁盘空间。

###3. vmdk 

vmdk是vmware的镜像格式。如要从vmware将虚拟机迁移到OpenStack环境，首先在虚拟机关机前提下，拷贝虚拟机镜像到openstack平台中，目标节点需要预留足够的存储空间。然后将vmdk镜像格式转换为raw格式。

```bash
# qemu-img convert -O raw xxx-disk.vmdk xxx-disk.raw
```
###4. vdi
vdi是VirtualBox支持的镜像格式。同样，如要从VirtualBox将虚拟机迁移到OpenStack环境，需要将vdi镜像格式转换为raw格式。

```bash
# qemu-img convert -O raw xxx-disk.vdi xxx-disk.raw
```

## 参考文档

1. Image Format：http://www.cnblogs.com/feisky/archive/2012/07/03/2575167.html
2. RedHat Understanding Virtual Disks：https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Virtualization/3.3/html/Administration_Guide/Understanding_virtual_disks.html
3. qcow2 snapshot_handout：https://kashyapc.fedorapeople.org/virt/lc-2012/snapshots-handout.html
4. qcow2性能：https://fedoraproject.org/wiki/Features/KVM_qcow2_Performance
5. vmdk：https://www.vmware.com/support/developer/vddk/vmdk_50_technote.pdf
