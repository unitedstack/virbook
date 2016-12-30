### Xen

Xen 起源于英国剑桥大学的一个研究项目，逐渐发展成为一个开源软件项目，发展非常迅速。
从技术的角度说，Xen基于混合模式，特权操作系统起到了宿主机操作系统的很多管理功能，其它非特权的虚拟机运行用户的程序。Xen最初是基于类虚拟化来实现的，通过修改Linux内核，实现处理器和内存的初始化，通过引入I/O的前端/后端驱动架构实现设备的虚拟化，利用类虚拟化的优势，Xen可以达到接近于物理机的性能。
![](/images/introduction/solutions/Xen.png)

作为开源软件X,Xen的主要特点有：

- 可移植性非常好，开发者可以将其移植到其他平台，也可以将其修改用于研究项目。
- 独特的类虚拟化支持，提供了接近于物理机的性能。但Xen的易用性和其它成熟的商业产品有一定的差距。

从技术的角度说，Xen基于混合模式，特权操作系统起到了宿主机操作系统的很多管理功能，其它非特权的虚拟机运行用户的程序。Xen最初是基于类虚拟化来实现的，通过修改Linux内核，实现处理器和内存的初始化，通过引入I/O的前端/后端驱动架构实现设备的虚拟化，利用类虚拟化的优势，Xen可以达到接近于物理机的性。

Xen包含三大部分：
- Xen Hypervisor: 直接运行于硬件之上是Xen客户操作系统与硬件资源之间的访问接口(如：)。通过将客户操作系统与硬件进行分类，Xen管理系统可以允许客户操作系统安全，独立的运行在相同硬件环境之上。
- Domain 0: 运行在Xen管理程序之上，具有直接访问硬件和管理其他客户操作系统的特权的客户操作系统。
- DomainU: 运行在Xen管理程序之上的普通客户操作系统或业务操作系统，不能直接访问硬件资源（如：内存，硬盘等），但可以独立并行的存在多个。

####Xen Hypervisor
Xen Hypervisor是直接运行在硬件与所有操作系统之间的基本软件层。它负责为运行在硬件设备上的不同种类的虚拟机（不同操作系统）进行CPU调度和内存分配。Xen Hypervisor对虚拟机来说不单单是硬件的抽象接口，同时也控制虚拟机的执行，让他们之间共享通用的处理环境。
Xen Hypervisor不负责处理诸如网络、外部存储设备、视频或其他通用的I/O处理。

####Domain 0
Domain 0 是经过修改的Linux内核，是运行在Xen Hypervisor之上独一无二的虚拟机，拥有访问物理I/O资源的特权，并且可以与其他运行在Xen Hypervisor之上的其他虚拟机进行交互。所有的Xen虚拟环境都需要先运行Domain 0，然后才能运行其他的虚拟客户机。
Domain 0 在Xen中担任管理员的角色，它负责管理其他虚拟客户机。
在Domain 0中包含两个驱动程序，用于支持其他客户虚拟机对于网络和硬盘的访问请求。这两个驱动分别是Network Backend Driver和Block Backend Driver。
Network Backend Driver直接与本地的网络硬件进行通信，用于处理来自Domain U客户机的所有关于网络的虚拟机请求。根据Domain U发出的请求Block Backend Driver直接与本地的存储设备进行通信然后，将数据读写到存储设备上。
![](/images/introduction/solutions/Xen_domain0.png)

####Domain U
Domain U客户虚拟机没有直接访问物理硬件的权限。所有在Xen Hypervisor上运行的半虚拟化客户虚拟机（简称：Domain U PV Guests）都是被修改过的基于Linux的操作系统、Solaris、FreeBSD和其他基于UNIX的操作系统。所有完全虚拟化客户虚拟机（简称：Domain U HVM Guests）则是标准的Windows和其他任何一种未被修改过的操作系统。
无论是半虚拟化Domain U还是完全虚拟化Domain U，作为客户虚拟机系统，Domain U在Xen Hypervisor上运行并行的存在多个，他们之间相互独立，每个Domain U都拥有自己所能操作的虚拟资源（如：内存，磁盘等）。而且允许单独一个Domain U进行重启和关机操作而不影响其他Domain U。

![](/images/introduction/solutions/Xen_hypervisor.png)

###参考资料：

Xen的工作原理：http://www.searchvirtual.com.cn/showcontent_25874.htm

kvm与Xen的区别：www.ibm.com/developerworks/cn/linux/l-using-kvm/
