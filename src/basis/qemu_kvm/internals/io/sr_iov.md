# SR-IOV 概述

PCI 透传技术供了非常快的 I/O。 但是，它阻止了共享 I/O 设备。 SR-IOV提供了一种单根根函数的机制，使得某些 I/O 设备（例如单个以太网端口）可以表现为多个单独的物理设备。

支持 SR-IOV 的设备可以被配置为多种功能出现在 PCI 配置空间中，每个功能都有其自己的配置空间与基地址寄存器。 VMM 通过将 VF 的实际配置空间映射到由 VMM 呈现给虚拟机的配置空间来向 VM 分配一个或多个VF。参见下图：

![](/images/basis/sr_iov.png)

支持 SR-IOV 的设备提供可配置数量的独立 VF，每个 VF 具有自己的 PCI 配置空间。 VMM 向虚拟机分配一个或多个 VF。 而 Intel VT-x 和 Intel VT-d 中的内存转换技术提供了硬件辅助技术，允许直接 DMA 传输到 VM，从而绕过 VMM 中的软件开关。

## SR-IOV 目标

SR-IOV 规范的目标是通过为每个虚拟机提供独立的内存空间，中断和 DMA 流来规避绕过 VMM 参与数据流移动的方式。SR-IOV 架构允许设备支持多个虚拟功能 (VF)。同时也尽可能减少设备增加 SR-IOV 功能而带来的额外的物理开销。

SR-IOV 引入了两种新的函数类型：
- 物理功能 (PF)：具有完备的 PCIe 功能，包括 SR-IOV 扩展与管理功能，可以理解为物理 PCI 设备的一个完整抽象，比如一块物理网卡。
- 虚拟功能 (VF)：“轻量级” PCIe 功能，物理设备功能的一个子集，用来完成特定功能，类比于物理网卡虚拟出的一块虚拟网卡。

## 参考资料

Intel An Introduction to SR-IOV Technology：http://www.intel.com/content/www/us/en/pci-express/pci-sig-sr-iov-primer-sr-iov-technology-paper.html
