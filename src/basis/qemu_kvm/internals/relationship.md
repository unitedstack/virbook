### QEMU

QEMU 是一套由 Fabrice Bellard 所编写的模拟处理器的自由软件，其本身就是一套完整的虚拟化解决方案，提供完全虚拟化功能。它与 Bochs，PearPC 近似，但其具有某些后两者所不具备的特性，如高速度及跨平台的特性，QEMU 可以虚拟出不同架构的虚拟机，如在 x86 平台上可以虚拟出 power 机器。KQEMU 为 QMEU 的加速器，经由 KQEMU 这个开源的加速器，QEMU 能模拟至接近真实电脑的速度。

QEMU本身可以不依赖于KVM，但是如果有 KVM 的存在并且硬件(处理器)支持比如 Intel VT 功能，那么QEMU在对处理器虚拟化这一块可以利用 KVM 提供的功能来提升性能。

### KVM

准确来说，KVM 是 Linux Kernel 的一个模块。可以用命令`modprobe`去加载 KVM 模块。加载了模块后，才能进一步通过其他工具创建虚拟机。

### QEMU+KVM

但仅有 KVM 模块是远远不够的，因为 KVM 模块只能够提供 CPU，Memory 的虚拟化功能，缺少 I/O 相关的虚拟化实现，并且用户无法直接控制内核模块去作事情。也就是说，KVM 缺少一套用户态的管理工具，所以还必须有一个运行在用户空间的工具才行。这个用户空间的工具，KVM 开发人员选择了已经成型的开源虚拟化软件 QEMU。说起来 QEMU 也是一套完整的开源全虚拟化解决方案。它的特点是可虚拟出不同的 CPU。比如说在 x86 的 CPU 上可虚拟一个 Power 的 CPU，并可利用 它编译出可运行在 Power 上的程序。KVM 使用了 QEMU 的一部分，并稍加改造，就成了可控制 KVM 虚拟机的用户空间工具—— KVM+QEMU。官方提供的 KVM 下载有两大部分(QEMU 和 KVM)三个文件：包括 KVM 模块、QEMU 工具以及二者的合集 (QEMU + KVM)。也就是说，你可以只升级 KVM 模块，也可以只升级 QEMU 工 具。这就是 KVM 和 QEMU 的关系。

KVM 与 QEMU的关系可以用下图来表示：
![](/images/basis/qemu_with_kvm.png)
