### PCI 透传简介
某些硬件资源可以轻松虚拟化，比如处理器和存储器；而另一些硬件资源则不然，比如视频适配器和串口。当共享不可能或没用时，Peripheral Component Interconnect (PCI) 透传技术提供有效使用这些资源的方法。

PCI 透传功能允许将来自物理机的 PCI 设备直接分配给虚拟机使用。虚拟机中的设备驱动驱动器可以直接驱动该硬件设备，而不依赖于来自物理机的任何驱动器能力。

设备共享是有代价的。无论设别模拟是在管理程序还是在一个独立 VM 中的用户空间中执行，都存在开销。只要有多个客户操作系统需要共享这些设备，这个开销就是值得的。如果共享不是必须的，则有更有效的方法来共享这些设备。因此，在最高层面上，设备透传就是向一个特定客户操作系统提供一种设备隔离，以便该设备能够被那个客户操作系统独占使用。

![](/images/basis/pci_passthrough.gif)

### PCI 透传的硬件支持

Intel 和 AMD 都在它们的新一代处理器架构中提供对设备透传的支持（以及辅助管理程序的新指令）。Intel 将这种支持称为 Virtualization Technology for Directed I/O (VT-d)，而 AMD 称之为 I/O Memory Management Unit (IOMMU)。不管是哪种情况，新的 CPU 都提供将 PCI 物理地址映射到客户虚拟系统的方法。当这种映射发生时，硬件将负责访问（和保护），客户操作系统在使用该设备时，就仿佛它不是一个虚拟系统一样。除了将客户机映射到物理内存外，新的架构还提供隔离机制，以便预先阻止其他客户机（或管理程序）访问该内存。

另一种帮助将中断缩放为大量 VM 的技术革新称为 Message Signaled Interrupts (MSI)。MSI 将中断转换为更容易虚拟化的消息（缩放为数千个独立中断），而不是依赖将被关联到一个客户机的物理中断 pin。从 PCI 2.2 开始，MSI 就已经可用，但 PCI Express (PCIe) 也提供 MSI，在 PCIe 中，MSI 支持将结构缩放为多个设备。MSI 是理想的 I/O 虚拟化技术，因为它支持多个中断源的隔离（而不是必须通过软件多路传输或路由的物理 pin）。

### 设备透传存在的问题

使用 PCI 透传时有一些注意事项。当 PCI 设备直接分配给虚拟机时，如果虚拟机不支持将设备热拔出，则无法迁移。此外，libvirt 不保证直接设备分配是安全的，将安全策略决策留给底层虚拟化技术。安全 PCI 设备通路通常需要特殊的硬件功能，例如用于Intel芯片组的VT-d特性或用于AMD芯片组的IOMMU。

### libvirt 对 PCI 透传的支持
有两种模式可以连接PCI设备，“管理”或“非管理”模式，但在写入时只有 KVM 支持“管理”模式连接。在受管模式下，配置的设备将在客户机启动时自动从主机操作系统驱动程序分离，然后在客户机关闭时重新连接。在非受管模式下，设备必须在引导虚拟机之前手动显式分离。如果设备仍然连接到主机操作系统，则客户机将拒绝启动。 libvirt 的`Node Device API`提供了从主机驱动程序分离以及重新连接 PCI 设备的方法。或者，物理主机可以被配置为将用于虚拟机的 PCI 设备列入黑名单，使他们不能物理主机不回去去加载相关设备的驱动程序。

在这两种模式下，虚拟化技术将始终在启动虚拟机之前以及虚拟机关闭之后，在设备上执行复位操作。这对于确保物理主机和虚拟机操作系统之间的隔离至关重要。一些复位技术作用于单个设备，而其他一些复位技术可以一次控制多个设备。在后一种情况下，需要将所有控制设备共同分配给同一虚拟机，否则将无法安全地重置。`Node Device API`可以用于通过手动地分离设备然后尝试执行复位操作来确定设备是否需要被共同分配。如果操作成功，则可以将设备自己分配给虚拟机。如果操作失败，那么就有必要在相同的 PCI 总线上共同分配该设备。

下面的代码检查 PCI 设备（由virNodeDevicePtr对象实例表示）是否可以被重置，并且因此得出是否可分配给虚拟机。
```c
virNodeDevicePtr dev = ....get virNodeDevicePtr for the PCI device...

if (virNodeDeviceDettach(dev) < 0) {
    fprintf(stderr, "Device cannot be dettached from the host OS drivers\n");
    return;
}

if (virNodeDeviceReset(dev) < 0) {
    fprintf(stderr, "Device cannot be safely reset without affecting other devices\n");
    return;
}

fprintf(stderr, "Device is suitable for passthrough to a guest\n");
```

### 利用 livbvirt 将 PCI 设备挂载到虚拟机上
在虚拟机的 xml 文件中，用`hostevice`标签来将一个 PCI 设备挂载到虚拟机。
```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000'
             bus='0x06'
             slot='0x12'
             function='0x5'/>
  </source>
</hostdev>
```
- 查询物理机上的 PCI 设备
```bash
$ virsh nodedev-list --tree
$ virsh nodedev-list | grep pci
$ lspci -n
```
- 查询相关 PCI 设备的详细信息
```bash
$ virsh nodedev-dumpxml pci_0000_ff_17_6
<device>
  <name>pci_0000_ff_17_6</name>
  <path>/sys/devices/pci0000:ff/0000:ff:17.6</path>
  <parent>computer</parent>
  <capability type='pci'>
    <domain>0</domain>
    <bus>255</bus>
    <slot>23</slot>
    <function>6</function>
    <product id='0x2fba'>Xeon E5 v3/Core i7 DDRIO (VMSE) 2 &amp; 3</product>
    <vendor id='0x8086'>Intel Corporation</vendor>
  </capability>
</device>
```

- 尝试从物理机上卸载该设备
```bash
virsh nodedev-dettach pci_0000_ff_17_6
# 如果不能成功卸载，则说明该 PCI 设备不能透传给虚拟机
setsebool -P virt_use_sysfs 1
```

- 将 PCI 信息中的十进制数字转换为十六进制，本文中如 bus = 255， slot = 23 ， function = 6 
```bash
$ printf %x 255
ff
$ printf %x 23
17
$ printf %x 6
6
```

- 创建 pci.xml 文件如下
```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000'
             bus='0xff'
             slot='0x17'
             function='0x6'/>
  </source>
</hostdev>
```

- 将设备挂载到虚拟机
```bash
virsh attach-device --domain instance_name --file pci.xml --live --persistent
```

参考：

[libvirt pci passthrough](https://libvirt.org/guide/html/Application_Development_Guide-Device_Config-PCI_Pass.html)

[IBM developer](https://www.ibm.com/developerworks/cn/linux/l-pci-passthrough/)

[redhat docs](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/5/html/Virtualization/chap-Virtualization-PCI_passthrough.html)