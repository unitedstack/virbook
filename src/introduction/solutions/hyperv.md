### Hyper-V

Hyper-V是微软的一款虚拟化产品，是微软第一个采用类似Vmware和Citrix开源Xen一样的基于hypervisor的技术。

Hyper-V采用微内核的架构，兼顾了安全性和性能的要求。Hyper-V底层的Hypervisor运行在最高的特权级别下，微软将其称为ring -1（而Intel则将其称为root mode），而虚拟机的OS内核和驱动运行在ring 0，应用程序运行在ring 3下，这种架构就不需要采用复杂的BT（二进制特权指令翻译）技术，可以进一步提高安全性。


####高效率的VMbus架构
由于Hyper-V底层的Hypervisor代码量很小，不包含任何第三方的驱动，非常精简，所以安全性更高。Hyper-V采用基于VMbus的高速内存总线架构，来自虚机的硬件请求（显卡、鼠标、磁盘、网络），可以直接经过VSC，通过VMbus总线发送到根分区的VSP，VSP调用对应的设备驱动，直接访问硬件，中间不需要Hypervisor的帮助。
这种架构效率很高，不再像以前的Virtual Server，每个硬件请求，都需要经过用户模式、内核模式的多次切换转移。更何况Hyper-V现在可以支持Virtual SMP，Windows Server 2008虚机最多可以支持4个虚拟CPU；而Windows Server 2003最多可以支持2个虚拟CPU。每个虚机最多可以使用64GB内存，而且还可以支持X64操作系统。

####完美支持Linux系统
Hyper-V可以很好地支持Linux，可以安装支持Xen的Linux内核，这样Linux就可以知道自己运行在 Hyper-V之上，还可以安装专门为Linux设计的Integrated Components，里面包含磁盘和网络适配器的VMbus驱动，这样Linux虚机也能获得高性能。
和之前的Virtual PC、Virtual Server类似，Hyper-V也是微软的一种虚拟化技术解决方案，但在各方面都取得了长足的发展。
Hyper-V可以采用半虚拟化（Para-virtualization）和全虚拟化（Full-virtualization）两种模拟方式创建虚拟机。半虚拟化方式要求虚拟机与物理主机的操作系统（通常是版本相同的Windows）相同，以使虚拟机达到高的性能；全虚拟化方式要求CPU支持全虚拟化功能（如Inter-VT或AMD-V），以便能够创建使用不同的操作系统（如Linux和Mac OS）的虚拟机。
从架构上讲Hyper-V只有“硬件－Hyper-V－虚拟机”三层，本身非常小巧，代码简单，且不包含任何第三方驱动，所以安全可靠、执行效率高，能充分利用硬件资源，使虚拟机系统性能更接近真实系统性能。


除了在构架上进行改进之外，Hyper-V还具有其它一些变化：Hyper-V基于64位系统：微软的新一代虚拟化技术Hyper-V是基于64位系统的，我们知道，32位系统的内存寻址空间只有4GB，在4GB的系统上再进行服务器虚拟化在实际应用中没有太大的实际意义。在支持大容量内存的64位服务器系统中，应用Hyper-V虚拟出多个应用才有较大的现实意义。微软上一代虚拟化产品Virtual Server和Virtual PC则是基于32位系统的。
硬件支持上大大提升：Hyper-V支持4颗虚拟处理器，支持64GB内存，并且支持x64操作系统；而Virtual Server只支持2个虚拟处理器，并且只能支持x86操作系统。并且在Hyper-V中还支持VLAN功能。支持Hyper-V服务器虚拟化需要启用了Intel-VT或AMD-V特性的x64系统。Hyper-V基于微内核Hypervisor架构，是轻量级的。Hyper-V中的设备共享架构，支持在虚拟机中使用两类设备：合成设备和模拟设备。
Hyper-V提供了对许多用户操作系统的支持:Novell SUSE Linux Enterprise Server 10 SP1、Windows VistaSP1 (x86)和Windows XP SP3 (x86)、windows XP SP2 (x64)。

###参考资料：
Hyper-V overview: https://technet.microsoft.com/library/hh831531

官网：https://www.microsoft.com/en-us/search/result.aspx?q=Hyper-V&form=MSHOME
