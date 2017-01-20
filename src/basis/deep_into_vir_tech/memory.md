# 内存虚拟化

- 对物理内存的两个基本认知


    1. 内存都是从物理地址 0 开始的

    2. 内存都是连续的，或者说至少在一些大的粒度上连续


- 内存虚拟化

    基于以上的认知，VMM 必须欺骗客户机，将物理机上的一段不从 0 开始的物理地址空间映射成虚拟机中一段从 0 开始并且连续的地址空间。因此 VMM 需要处理一下两个问题：

    1. 给定一个虚拟机，维护客户物理机地址到宿主机物理地址之间的映射关系。

    2. 截获虚拟机对客户机物理地址的访问，并根据所记录的映射关系，并将其转换成宿主机物理地址。

客户机虚拟地址到宿主机的物理地址至少要经过两次转换，如下图所示：

![](/images/basis/virtual_memory.jpg)

## 参考文档

wiki 内存虚拟化：https://en.wikipedia.org/wiki/Memory_virtualization