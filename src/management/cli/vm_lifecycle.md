# 虚拟机生命周期管理

这篇文章主要是从libvirt和OpenStack Nova的角度来讲述虚拟机生命周期的管理，包括虚拟机的创建、运行、暂停、删除等操作。

##基本概念

###一、domain的配置文件

在libvirt中，通常将虚拟机称为一个domain。我们可以看到在计算节点的/var/lib/nova/instances/instance\_uuid目录下有一个libvirt.xml文件，该文件描述了与该instance相关的所有信息，包括domain, network, storage, device等等。大体框架如下：

```
<domain type='kvm'>
   <name>test</name>
   ...
   <devices>
      ...
      <disk type='network' device='disk'> ... </disk>
      <disk type='network' device='cdrom'> ... </disk>
      <interface type='bridge'> ... </interface>
      <input type='tablet' bus='usb'/>
      ...
   </devices>
</domain>
```

###二、短暂性guest domain和持久性guest domain

libvirt区分两种不同的domain：短暂性的（transient）和持久性的（persistent\):

* 短暂性的domain会在domain关机或或者宿主重启之后将不会存在
* 持久性domain将会一直存在

不管是哪种类型的domain的，在创建时，它们的状态可以保存到文件中（`virsh dumpxml <domain>  >  **.xml`）。之后，只要该文件存在，这个domain就可以无数次从该文件中恢复（`virsh create/define <file>`）。所以，即使是短暂性的domain，也可以反复地从文件中恢复。

短暂性的domain和持久性的domain在创建过程上是有些不一样的。在启动持久性domain之前，需要对其定义（define）。对于短暂性的domain来说，创建和启动是一次性完成的。操作这两种类型的domain的命令也是有区别的。

在动态创建和销毁短暂性domain之前，该domain其他所有组件（比如  存储、网络、设备等）必须已经存在。

###三、domain的状态

一个gust domain可以处于一下状态：

* undefined（未定义）：这是一个起始状态，libvirt不会知道任何有关该domain的信息，该domain还没有被定义或创建。
* defined\(定义的）or stoped\(暂停的）：该domain已经定义了，但是尚不在运行中，也叫做暂停状态。只有持久性domain才会有这个状态，因为短暂性domain在暂停后就不会存在了。
* running\(正在运行中的\)：该domain已经被创建并且已在运行中，两种类型的domain都可以处于这种状态。
* paused（挂起了的）：该domain被hypervisor挂起，domain信息被临时地保存起来直到再次被继续（resume）。
* saved（保存了的）：与paused类似，不过domain的状态被保存到永久cunchu介质上（比如磁盘），因次它可以再次恢复。

各种状态之间的转换如下图所示：![](/images/management/domain_state.png)

从上面的图可以看出，在创建短暂性的domain时候，状态直接从undefined变成running；而在创建持久性domain时，需提前定义然后再启动，状态由undefined变为defined然后是running。

## 基本操作

###1. 定义domain

在运行domain之前，我们必须定义它。有两种方式可以用来创建domain，一种是使用命令行，另一种则是通过xml定义。以下属于命令行的创建方式：

```bash
# virt-install \           
             --virt-type kvm \
             --name MyNewVM \
             --ram 512 \
             --disk path=/var/lib/libvirt/images/MyNewVM.img,size=8 \
             --vnc \
             --cdrom /var/lib/libvirt/images/Fedora-14-x86_64-Live-KDE.iso \
             --network network=default,mac=52:54:00:9c:94:3b \
             --os-variant fedora14
```

上述命令会创建一个名为 MyNewVM的虚拟机，内存大小为512m，并指定其存储以及网络等组件。此外，我们也可以通过xml配置文件来创建虚拟卷和虚拟机。

xml文件定义domain的方式：

```
<domain type='kvm'>
    <name>test</name>
    <title>test</title>
    <description>test</description>
    <metadata></metadata>

    <memory unit='KiB'>8388608</memory>
    <currentMemory unit='KiB'>8388608</currentMemory>
    <vcpu placement='static'>4</vcpu>

    <os>
        <type arch='x86_64' >hvm</type>
        <boot dev='hd'/>
        <boot dev='cdrom'/>
    </os>

    <features>
        <acpi/>
        <apic/>
        <pae/>
    </features>

    <clock offset='utc'/>

    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>destroy</on_crash>

    <devices>

        <disk type='file' device='disk'>
            <source file='/var/lib/libvirt/images/test.raw'/>
            <target dev='sda' bus='scsi'/>
        </disk>

        <interface type='bridge'>
            <source bridge='br0'/>
            <model type='virtio'/>
        </interface>

    <graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0' passwd='123'> 
        <listen type='address' address='0.0.0.0'/>
    </graphics>

    </devices>
</domain>
```

### 2. 编辑domain

刚刚定义了一个虚拟机，若我们现在想修改某些配置，可以使用以下命令：

```bash
# virsh edit <domain>
```

### 3. 启动domain

当我们创建完虚拟机的xml文件之后，我们可以通过以下命令启动虚拟机：

```bash
# virsh create <file>                                               创建短暂性domain
# virsh define <file> ; virsh start <domain-name or domain-uuid>    创建持久性domain
```

注意：domain-id是随机分配的，关机后就没有了，所以启动domain只能用domain-name或者domain-uuid。


### 4. 停止和重启domain

```bash
# virsh shutdown <domain>
```

对于持久性domain来说，我们可以通过如下方式重启domain：

```bash
# virsh reboot <domain>
```

**注意：**kvm目前不支持reboot命令，可以通过先shutdown再start启动虚拟机。另外，我们不能停止一个短暂性的domain，一旦停止它，它就会处于一个undefined状态。

强制停止，相当于直接扒电源，一般在shutdown操作失败的情况下进行：

```bash
# virsh destory <domain>
```

### 5. 启动domain

```bash
# virsh start <domain>
```

### 6. 挂起domain

挂起domain，是虚拟机处于暂停状态，不可做一切操作。

```bash
# virsh suspend <domain>
```

### 7. 恢复被挂起的domain

恢复被挂起的domain，是虚拟机处于active状态。

```bash
# virsh resume <domain>
```

## 参考文档

1. libvirt-VM_lifecycle：http://wiki.libvirt.org/page/VM_lifecycle



