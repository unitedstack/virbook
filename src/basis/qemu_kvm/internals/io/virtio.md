### 概述

KVM 是必须使用硬件虚拟化辅助技术（如Intel VT-x、AMD-V）的 hypervisor，在 CPU 运行效率方面有硬件支持，其效率是比较高的。在有 Intel EPT 特性支持的平台上，内存虚拟化的效率也较高。QEMU+KVM 提供了全虚拟化环境，可以让客户机不经过任何修改就能运行在 KVM 环境中。

不过，KVM 在 I/O 虚拟化方面，传统的方式是使用 QEMU 纯软件的方式（全虚拟化）来模拟 I/O 设备（如网卡、磁盘、显卡等等），其效率并不非常高。在 KVM 中，可以在客户机中使用半虚拟化驱动 (Paravirtualized Drivers，PV Drivers) 来提高客户机的性能（特别是 I/O 性能）。目前，KVM 中实现 I/O 半虚拟化驱动的方式是采用了 RedHat 开发的 virtio 这个 Linux 上的设备驱动标准框架。virtio 驱动抽象架构如下：

![](/images/basis/virtio_abs.gif)


### 传统 QEMU I/O 模拟

QEMU 对物理机的 I/O 设备的模拟用下图来表示：

![](/images/basis/qemu_emulated_io.jpg)

使用 QEMU 模拟 I/O 的情况下，当客户机中的设备驱动程序 (device driver) 发起 I/O 操作请求之时，KVM 模块中的 I/O 操作捕获代码会拦截这次 I/O 请求，然后经过处理后将本次 I/O 请求的信息存放到 I/O 共享页，并通知用户控件的 QEMU 程序。QEMU 模拟程序获得 I/O 操作的具体信息之后，交由硬件模拟代码来模拟出本次的 I/O 操作，完成之后，将结果放回到 I/O 共享页，并通知 KVM 模块中的 I/O 操作捕获代码。最后，由 KVM 模块中的捕获代码读取 I/O 共享页中的操作结果，并把结果返回到客户机中。当然，这个操作过程中客户机作为一个 QEMU 进程在等待 I/O 时也可能被阻塞。另外，当客户机通过 DMA (Direct Memory Access) 访问大块 I/O 之时，QEMU 模拟程序将不会把操作结果放到 I/O 共享页中，而是通过内存映射的方式将结果直接写到客户机的内存中去，然后通过 KVM 模块告诉客户机 DMA 操作已经完成。

QEMU 模拟 I/O 设备的方式，其优点是可以通过软件模拟出各种各样的硬件设备，包括一些不常用的或者很老很经典的设备，而且它不用修改客户机操作系统，就可以实现模拟设备在客户机中正常工作。在 KVM 客户机中使用这种方式，对于解决手上没有足够设备的软件开发及调试有非常大的好处。而它的缺点是，每次 I/O 操作的路径比较长，有较多的 VMEntry、VMExit 发生，需要多次上下文切换 (context switch)，也需要多次数据复制，所以它的性能较差。

### virtio 架构

上面提到 virtio 是一个 Linux 设备驱动的框架，其高级架构如下图所示：

![](/images/basis/virtio_highlevel_arch.gif)

其中前端驱动（frondend，如 virtio-blk、virtio-net 等）是在虚拟机中存在的驱动程序模块，而后端处理程序 (backend) 是在 hypervisor 中实现的。在这前后端驱动之间，还定义了两层来支持客户机与 hypervisor 之间的通信。

其中，“virtio”这一层是虚拟队列接口，它在概念上将前端驱动程序附加到后端处理程序。一个前端驱动程序可以使用0个或多个队列，具体数量取决于需求。例如: virtio-net 网络驱动程序使用两个虚拟队列（一个用于接收，另一个用于发送），而 virtio-blk 块驱动程序仅使用一个虚拟队列。虚拟队列实际上被实现为跨越虚拟机操作系统和 hypervisor 的衔接点，但它可以通过任意方式实现，前提是虚拟机操作系统和 virtio 后端程序都遵循一定的标准，以相互匹配的方式实现它。而 virtio-ring 实现了环形缓冲区 (ring buffer)，用于保存前端驱动和后端处理程序执行的信息，并且它可以一次性保存前端驱动的多次 I/O 请求，并且交由后端去动去批量处理，最后实际调用宿主机中设备驱动实现物理上的 I/O 操作，这样做就可以根据约定实现批量处理而不是客户机中每次 I/O 请求都需要处理一次，从而提高虚拟机与 hypervisor 信息交换的效率。

virtio 半虚拟化驱动的方式，可以获得很好的 I/O 性能，其性能几乎可以达到和 native（即：非虚拟化环境中的原生系统）差不多的 I/O 性能。所以，在使用 KVM 之时，如果宿主机内核和客户机都支持 virtio 的情况下，一般推荐使用 virtio 达到更好的性能。当然，virtio 的也是有缺点的，它必须要客户机安装特定的 virtio 驱动使其知道是运行在虚拟化环境中，且按照 virtio 的规定格式进行数据传输，不过客户机中可能有一些老的 Linux 系统不支持 virtio 和主流的 Windows 系统需要安装特定的驱动才支持 virtio。不过，较新的一些Linux发行版（如 RHEL 6.3、Fedora 17等）默认都将virtio相关驱动编译为模块，可直接作为客户机使用 virtio，而且对于主流 Windows 系统都有对应的 virtio 驱动程序可供下载使用。

### 总结

** virtio 与传统 QEMU I/O 设备模拟的性能差异，主要是由于 QEMU 传统 I/O 模拟，在与物理机相关的 I/O 设备进行数据交换时，操作路径比较长、复杂，需要 QEMU 多次捕获异常和触发异常，并且频繁的切换上下文，导致 I/O 性能较差。**

**而 virtio 的出现，一方面是为了 Linux 平台上出现的各种各样的 hypervisor 提供一个统一的 I/O 抽象层（框架）；另一方面提供一个简洁的、高效的数据交换通道，实现 I/O 设备的半虚拟化，让虚拟机能够以接近真实的性能进行 I/O 操作。**

### 参考链接
[http://smilejay.com/2012/11/virtio-overview/](http://smilejay.com/2012/11/virtio-overview/)

[virtio 白皮书](http://www.ozlabs.org/~rusty/virtio-spec/virtio-paper.pdf)

[virtio IBM](https://www.ibm.com/developerworks/cn/linux/l-virtio/)