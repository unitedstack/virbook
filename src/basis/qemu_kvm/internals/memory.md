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

### **影子页表**

影子页表完成了客户机虚拟地址GVA到宿主机物理地址HPA的直接转换。但是由于客户机中每个进程都有自己的虚拟地址空间，所以 KVM 需要为客户机中的每个进程页表都要维护一套相应的影子页表，影子页表维护虚拟地址（VA）到机器地址（MA）的映射关系。而客户机页表维护VA到客户机物理地址（GPA）的映射关系。当VMM捕获到客户机页表的修改后，VMM会查找负责GPA到MA映射的P2M页表或者哈希函数，找到与该GPA对应的MA，再将MA填充到真正在硬件上起作用的影子页表，从而形成VA到MA的映射关系。而Guest的页表则无需变动。

当客户机访问内存时，真正被装入宿主机内存管理单元（Memory ManagerMent Unit, MMU，负责虚拟地址和物理地址之间的映射）的是客户机当前页表所对应的影子页表。这样就完成了GVA到HPA的直接映射，大大提高了效率。

![](/images/basis/shadow_page_table.png)

### EPT

如上述所说，KVM 需要为客户机中的每个进程页表都要维护一套相应的影子页表，因此内存开销会非常的大，且实现方法比较复杂，因此引入了扩展页表EPT来解决上述问题。EPT是Intel提供的硬件技术，使客户机能在原有的页表的基础上，增加一个EPT页表，用于记录GPA到MA的映射关系。VMM预先把EPT页表设置到CPU中。Guest修改Guest页表，无需VMM干预。地址转换时，CPU自动查找两张页表完成Guest虚拟地址到机器地址的转换，从而降低整个内存虚拟化所需的开销。

![](/images/basis/ept.png)

