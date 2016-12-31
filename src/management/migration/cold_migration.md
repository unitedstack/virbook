# 离线迁移

在关机的情况下对KVM虚拟机进行迁移，将虚拟机从一个宿主机迁移到另外一个宿主机，迁移中，虚拟机一直处于关机状态。离线迁移适用于宿主机需要关机进行维修的情况，或停机比较长时间的场景。通常是计划内的。

对虚拟机执行离散迁移的操作相对比较简单，将虚拟机关机后，只需要迁移虚拟机的xml配置文件和磁盘文件，若使用了共享存储，只需迁移虚拟机的配置文件。

操作步骤如下：

1.关闭指定需要执行离散迁移的虚拟机，以虚拟机instance-00000717为例：

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh shutdown instance-00000717
```

查看该虚拟机状态：

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh list --all
 Id    Name                           State
----------------------------------------------------
 5     instance-00000719              running
 -     instance-000006a8              shut off
 -     instance-00000717              shut off     # 已关闭
 -     instance-0000071a              shut off
 -     instance-0000071c              shut off
```

2.下载该虚拟机的配置文件到717.xml：

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh dumpxml instance-00000717 > 717.xml
```

3.拷贝该虚拟机的配置文件到目的宿主机的对应目录下：

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ scp 717.xml root@10.0.103.69 /etc/libvirt/qemu
```

4.切换到目的宿主机控制台`/etc/libvirt/qemu`目录下，定义该xml文件：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ virsh define 717.xml 
Domain instance-00000717 defined from 717.xml
```

这里需要注意，由于目的主机不存在717.xml中指定的日志文件以及网桥，所以需要在目的宿主机中创建这些内容。

```
...
    <serial type='file'>
      <source path='/var/lib/nova/instances/36969fe2-2f19-43f8-994d-91a4a04b8abe/console.log'/>
      <target port='0'/>
    </serial>
    ...
    <interface type='bridge'>
      <mac address='fa:16:3e:bb:5a:be'/>
      <source bridge='qbrd0c34abd-b3'/>
      <target dev='tapd0c34abd-b3'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
```

5.在开启该虚拟机前，首先需要在目的宿主机上做如下准备操作：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ mkdir /var/lib/nova/instances/36969fe2-2f19-43f8-994d-91a4a04b8abe/
[root@server-69.103.hatest.ustack.in ~ ]$ touch console.log
[root@server-69.103.hatest.ustack.in ~ ]$ brctl addbr qbrd0c34abd-b35
```

6.开启虚拟机：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ virsh start instance-00000717
Domain instance-00000717 started
```

7.查看结果：

```bash
[root@server-69.103.hatest.ustack.in ~ ]$ virsh list
 Id    Name                           State
----------------------------------------------------
 32    node01                         running
 34    node02                         running
 37    test                           running
 57    instance-00000717              running
```

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh list --all
 Id    Name                           State
----------------------------------------------------
 5     instance-00000719              running
 6     instance-0000071a              running
 8     instance-0000071c              running
 -     instance-000006a8              shut off
 -     instance-00000717              shut off
```

离线迁移成功，但是依然可以看到源宿主机上该虚拟机依然存在，但是处于关机状态，我们可以执行`virsh undefine instance-00000717`将其彻底销毁。

