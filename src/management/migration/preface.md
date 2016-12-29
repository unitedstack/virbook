## Libvirt Migration

将虚拟机从当前的宿主机迁移到目的宿主机的过程，即虚拟机的迁移。迁移过程主要是将当前宿主机上指定需要迁移的虚拟机的内存，磁盘，设备状态等迁移到到另一台宿主机，并根据这些信息在目的宿主机上启动该虚拟机。

**一、虚拟机的迁移有如下好处**

* **负载均衡**

  一台主机超载时，其虚拟机的一个或多个可以即时迁移到其他主机上。

* **针对主机升级或进行更改**

  需要升级、添加或卸下一台主机的硬件设备时，虚拟机可以安全地转移到其他主机。这也就是说客机不会由于任何一个主机的更改而停机。

* **节能**

  虚拟机器可以重新分配至其他主机，空载主机系统可以在使用率低的时段关机以节省能源和开支。

* **地理迁移**

  为实现更低的延迟或其他特殊情况，虚拟化机器可以转移到另一个物理位置。

**二、虚拟机的迁移有如下两种方式**

* 离线迁移

  当虚拟机处于关机状态时，针对此虚拟机的迁移操作为离线迁移。
* 动态迁移

  将虚拟机处于active状态时，针对此虚拟机的迁移操作为动态迁移。

后面篇章将详细介绍KVM虚拟机在两个物理宿主机上迁移的实现过程。

### 参考资料

1. [https://www.ibm.com/developerworks/cn/linux/l-cn-mgrtvm1/](https://www.ibm.com/developerworks/cn/linux/l-cn-mgrtvm1/)
2. [Red Hat Enterprise Linux - virtualization](https://access.redhat.com/documentation/zh-CN/Red_Hat_Enterprise_Linux/7/html/Virtualization_Getting_Started_Guide/sec-migration.html#sec-benefits_of_migrating)



