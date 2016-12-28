### 概述

KVM 是必须使用硬件虚拟化辅助技术（如Intel VT-x、AMD-V）的 hypervisor，在 CPU 运行效率方面有硬件支持，其效率是比较高的。在有 Intel EPT 特性支持的平台上，内存虚拟化的效率也较高。QEMU+KVM 提供了全虚拟化环境，可以让客户机不经过任何修改就能运行在 KVM 环境中。不过，KVM 在 I/O 虚拟化方面，传统的方式是使用 QEMU 纯软件的方式（全虚拟化）来模拟 I/O 设备（如网卡、磁盘、显卡等等），其效率并不非常高。在 KVM 中，可以在客户机中使用半虚拟化驱动 (Paravirtualized Drivers，PV Drivers) 来提高客户机的性能（特别是 I/O 性能）。目前，KVM 中实现 I/O 半虚拟化驱动的方式是采用了 virtio 这个 Linux 上的设备驱动标准框架。virtio 驱动抽象架构如下：

![](/images/basis/virtio_abs.gif)

### 传统 QEMU I/O 模拟



### 架构

上面提到 virtio 是一个 Linux 设备驱动的框架，其高级架构如下图所示：

![](/images/basis/virtio_highlevel_arch.gif)

其中前端驱动（frondend，如 virtio-blk、virtio-net 等）是在虚拟机中存在的驱动程序模块，而后端处理程序 (backend) 是在 hypervisor 中实现的。在这前后端驱动之间，还定义了两层来支持客户机与 hypervisor 之间的通信。

其中，“virtio”这一层是虚拟队列接口，它在概念上将前端驱动程序附加到后端处理程序。一个前端驱动程序可以使用0个或多个队列，具体数量取决于需求。例如: virtio-net 网络驱动程序使用两个虚拟队列（一个用于接收，另一个用于发送），而 virtio-blk 块驱动程序仅使用一个虚拟队列。虚拟队列实际上被实现为跨越虚拟机操作系统和 hypervisor 的衔接点，但它可以通过任意方式实现，前提是虚拟机操作系统和 virtio 后端程序都遵循一定的标准，以相互匹配的方式实现它。而 virtio-ring 实现了环形缓冲区 (ring buffer)，用于保存前端驱动和后端处理程序执行的信息，并且它可以一次性保存前端驱动的多次 I/O 请求，并且交由后端去动去批量处理，最后实际调用宿主机中设备驱动实现物理上的 I/O 操作，这样做就可以根据约定实现批量处理而不是客户机中每次 I/O 请求都需要处理一次，从而提高虚拟机与 hypervisor 信息交换的效率。

virtio 半虚拟化驱动的方式，可以获得很好的 I/O 性能，其性能几乎可以达到和 native（即：非虚拟化环境中的原生系统）差不多的 I/O 性能。所以，在使用 KVM 之时，如果宿主机内核和客户机都支持 virtio 的情况下，一般推荐使用 virtio 达到更好的性能。当然，virtio 的也是有缺点的，它必须要客户机安装特定的 virtio 驱动使其知道是运行在虚拟化环境中，且按照 virtio 的规定格式进行数据传输，不过客户机中可能有一些老的 Linux 系统不支持 virtio 和主流的 Windows 系统需要安装特定的驱动才支持 virtio。不过，较新的一些Linux发行版（如 RHEL 6.3、Fedora 17等）默认都将virtio相关驱动编译为模块，可直接作为客户机使用 virtio，而且对于主流 Windows 系统都有对应的 virtio 驱动程序可供下载使用。

