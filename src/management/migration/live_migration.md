## 在线迁移

在不关闭KVM虚拟机的情况下，将虚拟机从一个宿主机迁移到另外一台宿主机，整个迁移过程中，虚拟机一直在运行。在线迁移适用于宿主机需要重启，短暂关机的情况。通常都是在计划内的。

对虚拟机执行在线迁移的操作也很简单，且不需要在两个物理主机之间传送虚拟机的配置文件。使用virsh工具的话只需要一条命令即可，但是和离线迁移中需要注意的问题一样，需要为在目的宿主机上提前建立与虚拟机相关的日志文件和网桥。

操作步骤如下：

1.查看所有处于运行状态的虚拟机，选定测试虚拟机为instance-00000717：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ virsh list 
 Id    Name                           State
----------------------------------------------------
 32    node01                         running
 34    node02                         running
 37    test                           running
 57    instance-00000717              running
 61    instance-0000071c              running
```

1. 在目的宿主机上做如下准备操作：

```bash
   [root@server-68.103.hatest.ustack.in ~ ]$ mkdir /var/lib/nova/instances/36969fe2-2f19-43f8-994d-91a4a04b8abe/
   [root@server-68.103.hatest.ustack.in ~ ]$ touch console.log
   [root@server-68.103.hatest.ustack.in ~ ]$ brctl addbr qbrd0c34abd-b3
```

2. 执行在线迁移命令：

```bash
   [root@server-69.103.hatest.ustack.in ~ ]$ virsh migrate instance-00000717 --live qemu+ssh://10.0.103.68/system --unsafe
```

3. 查看结果，迁移成功：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ virsh list --all
 Id    Name                           State
----------------------------------------------------
 32    node01                         running
 34    node02                         running
 37    test                           running
 61    instance-0000071c              running
 -     instance-00000717              shut off
```

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh list
 Id    Name                           State
----------------------------------------------------
 5     instance-00000719              running
 13    instance-00000717              running
```

