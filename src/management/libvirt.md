## 简介

libvirt是一组软件的汇集，包括用C语言实现的API，一个守护进程（libvirtd），一个命令行工具（virsh）。该项目的目标是“提供一套**通用、稳定、安全**的虚拟化抽象层，用来管理**单个物理节点**上的虚拟机实例“以及其他虚拟化功能，为其他更高层的管理工具和应用程序提供统一的虚拟化功能操作接口。

## 功能介绍

libvirt的主要功能如下：

* **虚拟机管理：**管理虚拟机的生命周期，包括：虚拟机的provision、create、delete、start、stop、monitor、control、migrate。
* **远程连接管理：**用户可以在运行着 libvirtd的机器上执行libvirt的所有功能，包括远程机器。但并不是所有支持libvirt的hypervisor都需要libvirtd。
* **存储管理：**libvirt支持多种存储driver，任何运行了libvirtd的机器都可以用于管理多种类型的存储。
* **网络接口管理：**任何运行了libvirtd的主机都可以用来管理物理和逻辑的网络接口。
* **网络NAT和路由管理：**任何运行 libvirt 守护进程的主机都可以管理和创建虚拟网络。Libvirt 虚拟网络使用防火墙规则实现一个路由器，为虚拟机提供到主机网络的透明访问。

libvirt功能的更多详细内容请移步：[https://libvirt.org/docs.html](https://libvirt.org/docs.html)

libvirt基本架构请移步：[http://www.ibm.com/developerworks/cn/linux/l-libvirt/](http://www.ibm.com/developerworks/cn/linux/l-libvirt/)

## Libvirt与OpenStack Nova

Nova服务在OpenStack项目中的主要功能是提供大规模可扩展的，按需自服务使用计算资源，包括虚拟CPU，虚拟内存，虚拟网卡等。即主要是管理虚拟机的生命周期以及负责管理虚拟机需要使用的虚拟计算资源。

Nova中所有的计算资源都由Hypervisor来管理，对于某些类型的Hypervisor，如KVM，LXC，QEMU等Hypervisor来说，资源管理功能都是通过调用libvirt的统一接口来实现。

### 参考文档

1. [https://wiki.archlinux.org/index.php/libvirt\_\(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87](https://wiki.archlinux.org/index.php/libvirt_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)\)
2. [https://libvirt.org/docs.html](https://libvirt.org/docs.html)



