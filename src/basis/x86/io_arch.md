# x86 的 I/O 架构
I/O 是 CPU 访问外部设备的方法，设备通常通过寄存器和设备 RAM 将自身的功能展现给 CPU，CPU 读写这些寄存器和 RAM 即可完成对设备的访问和操作。x86 I/O 架构分为如下两类：

- Port I/O

    即端口 I/O，通过 I/O 端口访问设备寄存器。x86 有 65536 个 8 位的 I/O 端口，编号为 0x0 - 0xFFFF。如果把端口号看作访问端口设备的地址，那么这 65536 个端口就构成 64KB 的地址空间，称为 I/O 端口地址空间。I/O 端口地址空间是独立的，也就是说他并不是线性地址空间或者物理地址空间的一部分。

- MMIO (Memory Map I/O)

    即通过内存访问的形式访问设备寄存器或者 RAM。MMIO 与 Port I/O 最大的不同在于 MMIO 要占用 CPU 的物理地址空间。MMIO 是一种更加先进的 I/O 访问方式。

## DMA

DMA （直接内存访问）是将 CPU 从 I/O 操作中解放出来的一种技术。如果设备向内存复制数据都经过 CPU，则会消耗大量的 CPU 时间。通过 DMA，驱动程序可以事先（或者在需要的时候）设定一个内存地址，设备就可以绕开 CPU 直接向内存中复制（或读取数据）。

DAM 又分为：同步 DMA：是指 DMA 操作由软件发起，与异步 DMA：指 DMA 操作由设备发起。

设备的 DMA 操作都是使用物理地址访问内存，不经过线性地址到物理地址的转换。

与 DMA 相关的技术还有 IOMMU 这个会在 VT-d 技术中讲到。


## PCI 设备

具体内容请参看参考资料的相关内容。



## PCI Express

具体内容请参看参考资料中相关内容。

## 参考资料

Port I/O vs MMIO：http://superuser.com/questions/703695/difference-between-port-mapped-and-memory-mapped-access

PCI 系统详解：http://www.tldp.org/LDP/tlk/dd/pci.html

PCI Express: https://en.wikipedia.org/wiki/PCI_Express
