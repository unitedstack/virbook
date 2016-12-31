# KVM 内存虚拟化

对于单一宿主机来说，它有连续的，地址从0开始且可以访问的内存空间。若要满足在其上运行的多个客户机同样也有连续的，从0开始的内存空间地址，因此需要对物理内存进行虚拟化。

为了实现内存虚拟化，让客户机像宿主机一样有相同内存空间布局，就需要解决客户机中内存空间连续和地址从0开始的问题。KVM引入了一层新的地址空间——客户机物理地址空间（Guest Physical Address, GPA），因此，对客户机来说，就有了自己（虚拟）的物理地址。但实际上，GPA只是宿主机虚拟地址空间（Host Virtual Address, HVA）在客机地址空间GPA的一个映射，该映射由VMM来维护，本篇章中VMM以KVM为例。然而客户机会看到从0开始的连续地址空间，但是对宿主机来说，GPA并不一定是连续的，如下图所示：

![](/images/basis/memory.png)

KVM不仅维护着客户机物理地址到宿主机虚拟地址的映射，KVM还需要实现客户机虚拟地址（Guest Virtual Address, GVA）到宿主机物理地址（Host Physical Address, HPA）之间的转换。

以上提到了四种地址：

* 客户机虚拟地址（Guest Virtual Address, GVA）

* 客户机物理地址（Guest Physical Address, GPA）

* 宿主机虚拟地址（Host Virtual Address, HVA）

* 宿主机物理地址（Host Physical Address, HPA）

在传统的实现方案中，客户机虚拟地址GVA到宿主机物理地址HPA之间的转换，会依次通过如下几个步骤：

GVA  --&gt;  GPA  --&gt;  HVA  --&gt;  HPA

很明显会发现，以上传统方案需要对地址进行多次转换，而且需要KVM的介入，效率非常低。为提供GVA到HPA的地址转换效率，KVM提供了两种地址转换方式：

1. 基于纯软件的内存全虚拟化技术，通过影子页表 \(Shadow Page Table\) 来实现。
2. 基于硬件对虚拟化的支持，通过扩展页表EPT（Extended Page Table）来实现。

## **影子页表**

影子页表完成了客户机虚拟地址GVA到宿主机物理地址HPA的直接转换。但是由于客户机中每个进程都有自己的虚拟地址空间，所以 KVM 需要为客户机中的每个进程页表都要维护一套相应的影子页表，影子页表维护虚拟地址（VA）到机器地址（MA）的映射关系。而客户机页表维护VA到客户机物理地址（GPA）的映射关系。当VMM捕获到客户机页表的修改后，VMM会查找负责GPA到MA映射的P2M页表或者哈希函数，找到与该GPA对应的MA，再将MA填充到真正在硬件上起作用的影子页表，从而形成VA到MA的映射关系。而Guest的页表则无需变动。

当客户机访问内存时，真正被装入宿主机内存管理单元（Memory ManagerMent Unit, MMU，负责虚拟地址和物理地址之间的映射）的是客户机当前页表所对应的影子页表。这样就完成了GVA到HPA的直接映射，大大提高了效率。

![](/images/basis/shadow_page_table.png)

## 硬件支持的EPT页表

如上述所说，KVM 需要为客户机中的每个进程页表都要维护一套相应的影子页表，因此内存开销会非常的大，且实现方法比较复杂，因此引入了扩展页表EPT来解决上述问题。EPT是Intel提供的硬件技术，使客户机能在原有的页表的基础上，增加一个EPT页表，用于记录GPA到MA的映射关系。VMM预先把EPT页表设置到CPU中。Guest修改Guest页表，无需VMM干预。地址转换时，CPU自动查找两张页表完成Guest虚拟地址到机器地址的转换，从而降低整个内存虚拟化所需的开销。

## libvirt下获取虚机内存使用

在我们的云平台中，基本都需要这样一个功能，就是收集虚拟机监控数据，比如cpu使用率、内存使用率、磁盘io、网络io等信息。通常这些信息Hypervisor都会提供接口供你获取，这种获取方式成本是低廉的，通常不会对整个虚拟化环境有影响。如果想要获取更多的监控详情信息，那么则需要在虚机里面安装agent来收集监控数据，这种方式获取成本高，有时候用户可能不会接受镜像里面有agent的事实，这好比被安装了后门一样。两种方式各有优劣，看各自的需求场景，具体使用具体分析。

本文主要讨论是如何能通过libvirt接口获取虚拟机的内存使用量，主要是针对kvm虚拟化。

**获取接口**

使用libvirt的命令行工具可以获取虚机的内存信息，方式如下:

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh list
 Id    Name                           State
----------------------------------------------------
 7     instance-0000071c              running
 8     instance-00000719              running
 9     instance-00000717              running
 10    instance-0000071a              running
 11    instance-0000071d              running
 12    instance-0000071e              running

[root@server-68.103.hatest.ustack.in ~ ]$ virsh dommemstat instance-0000071c
actual 4194304
swap_in 0
rss 531256
```
actual是启动虚机时设置的最大内存，rss是qemu process在宿主机上所占用的内存。但是我们要获取的是虚机内部的内存使用情况，这样明显不能满足需求。

我们还需要给虚机做些配置，给虚机的libvirt.xml描述文件添加下面的内容：
```
#每10s钟收集一次
<memballoon model="virtio">
    <stats period="10"/>
</memballoon>
```
再次查询虚机的内存信息，得到：

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh dommemstat instance-0000071c
actual 4194304
swap_in 0
swap_out 0
major_fault 357
minor_fault 368181
unused 3796636
available 3924688
rss 578544
```
unused代表虚机内部未使用的内存量，available代表虚机内部识别出的总内存量，那么虚机内部的内存使用量则是(available-unused)的结果。

## 内存虚拟化的优化

KVM中提供了一种称为`ballooning`的模块作为一个伪设备驱动程序或内核服务载入到客户机系统中，该模块不为操作系统或上层应用程序提供被调用的接口，而只是为VMM提供了私有的交互接口，通过这个接口和VMM交互，然后宿主机系统内存使用的紧张程度来动态增加或回收客户机的内存占用。 如果你的云计算环境准备实施超售,那么这个机制是十分有用的，因为宿主机上的客户机不可能同时满载，这样便可以有效利用物理内存。

OpenStack环境中，内存的超售比由/etc/nova/nova.conf下的ram_allocation参数配置。该参数必须根据实际情况来合理设置，否则会出现内存负载过高，导致虚拟机被关机。

此外还有HugePage和Transparent HugePage技术。前者可以给客户机分配一块大内存独占使用，但是因为独占导致很多不灵活，不能在宿主机内存紧张的时候换出；而后者则是继承了HugePage的优点并弥补了这个缺点。大页技术的使用也需要慎重，如果客户机运行的应用比较依赖内存性能(Redis之流)，那么开启这个是值得的。

OpenStack环境中，HugePage由/etc/libvirt/qemu.conf配置文件下的hugetlbfs_mount参数来配置。

![](/images/basis/ept.png)


## 参考文档

1. [libvirt下获取虚机内存使用](http://niusmallnan.com/_build/html/_templates/openstack/libvirt_memory_usage.html)
2. [虚拟化技术原理](http://www.ywnds.com/?p=5856)
3. [内存虚拟化](http://blog.chinaunix.net/uid-26163398-id-5674852.html)
4. [内存虚拟化](http://udn.yyuap.com/thread-51437-1-1.html)
5. [KVM MMU Virtualization](http://events.linuxfoundation.org/slides/2011/linuxcon-japan/lcj2011_guangrong.pdf)
6. [KVM中的ballooning](http://www.programgo.com/article/87372681371/)
7. [KVM - Using Hugepages](https://help.ubuntu.com/community/KVM%20-%20Using%20Hugepages)
