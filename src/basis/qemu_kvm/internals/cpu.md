### KVM 中的 vCPU

KVM 是基于 CPU 辅助的全虚拟化方案，它需要CPU虚拟化特性的支持。因此创建 KVM 虚拟机，需要确定 CPU 开启了相应的虚拟化技术(Intel VT、AMD VSM)，一般在 BIOS 中设置。也可以通过 `egrep -c '(vmx|svm)' /proc/cpuinfo` 命令来确定是否支持 KVM，返回值 0 代表不支持 KVM。

qemu-kvm 通过对 `/dev/kvm` 的 一系列 `ICOTL` 命令控制虚机，一个 KVM 虚机即一个 Linux qemu-kvm 进程，与其他 Linux 进程一样被Linux 进程调度器调度。KVM 虚机包括虚拟内存、虚拟CPU和虚机 I/O设备，其中，内存和 CPU 的虚拟化由 KVM 内核模块负责实现，I/O 设备的虚拟化由 QEMU 负责实现。同时虚拟机的内存是 qemu-kvm 进程的内存地址空间中一部分。

KVM 虚拟中，每一个 vCPU 对应的是一个 qemu-kvm 进程的线程。由于 KVM 中对于 CPU 的虚拟化是利用 CPU 辅助实现的全虚拟化，因此每个 vCPU 指令实际上是直接在物理 CPU 上执行的。

几个概念：
- socket （颗，CPU 的物理单位）
- core （核，每个 CPU 中的物理内核）
- thread （超线程，通常来说，一个 CPU core 只提供一个 thread，这时客户机就只看到一个 CPU；但是，超线程技术实现了 CPU 核的虚拟化，一个核被虚拟化出多个逻辑 CPU，可以同时运行多个线程）

KVM 中，可以指定 socket，core 和 thread 的数目，比如 设置 `-smp 5,sockets=5,cores=1,threads=1`，则 vCPU 的数目为 5*1*1 = 5。客户机看到的是基于 KVM vCPU 的 CPU 核，而 vCPU 作为 QEMU 线程被 Linux 作为普通的线程/轻量级进程调度到物理的 CPU 核上。关于三个参数对虚拟机性能的影响可以参考[这里](http://frankdenneman.nl/2013/09/18/vcpu-configuration-performance-impact-between-virtual-sockets-and-virtual-cores/)。

详细请参考[这里](http://www.cnblogs.com/sammyliu/p/4543597.html)。

### CPU 拓扑

早期的时候，每台服务器都是单 CPU，随着技术发展，在硬件制造技术飞跃发展的今天，一个物理服务器上往往不止拥有一个物理 CPU，因此为了解决多 CPU 协同工作，CPU 拓扑这个概念应运而生。

NUMA (Non-Uniform Memory Access，非一致性内存访问)和 SMP (Symmetric Multi-Processor，对称多处理器系统)是两种不同的 CPU 硬件体系架构。

SMP 的主要特征是共享，所有的CPU共享使用全部资源，例如内存、总线和I/O，多个CPU对称工作，彼此之间没有主次之分，平等地访问共享的资源，这样势必引入资源的竞争问题，从而导致它的扩展内力非常有限。

NUMA 技术将 CPU 划分成不同的组（Node)，每个 Node 由多个 CPU 组成，并且有独立的本地内存、I/O等资源。Node 之间通过互联模块连接和沟通，因此除了本地内存外，每个 CPU 仍可以访问远端 Node 的内存，只不过效率会比访问本地内存差一些，我们用Node之间的距离（Distance，抽象的概念）来定义各个Node之间互访资源的开销。

我们可以通过`numactl --hardware`来查看 CPU 硬件信息：
    ```bash
    $ numactl --hardware
    available: 2 nodes (0-1)
    node 0 cpus: 0 2 4 6 8 10 12 14 16 18 20 22 24 26 28 30 32 34 36 38
    node 0 size: 32543 MB
    node 0 free: 196 MB
    node 1 cpus: 1 3 5 7 9 11 13 15 17 19 21 23 25 27 29 31 33 35 37 39
    node 1 size: 32768 MB
    node 1 free: 117 MB
    node distances:
    node   0   1
    0:  10  21
    1:  21  10
    ```
可以看出，该服务器上的 40 个逻辑 CPU 被分为两个 node。node0、node1分配的本地内存分别为 32543、32768 MB。

- KVM 配置 vCPU 的 NUMA 拓扑，虚拟机 xml 文件如下：
    ```xml
    <cpu>
        <numa>
            <cell id='0' cpus='0-3' memory='512000' unit='KiB'/>
            <cell id='1' cpus='4-7' memory='512000' unit='KiB' memAccess='shared'/>
        </numa>
    </cpu>
    ```
`cell` 标签与 NUMA 中的 node 概念对应。

- 也可以利用 `virsh` 命令来配置虚拟机的 vCPU NUMA 拓扑
```bash
virsh numatune <domain> [--mode <string>] [--nodeset <string>] [--config] [--live] [--current]
```

[参考](http://kodango.com/cpu-topology)



### CPU 亲和性（affinity）

软亲和性意味着进程并不会在处理器之间频繁迁移，而硬亲和性，意味着进程需要在您指定的处理器上运行。

简单地说，CPU 亲和性（affinity） 就是进程要在某个给定的 CPU 上尽量长时间地运行而不被迁移到其他处理器的倾向性。Linux 内核进程调度器天生就具有被称为 软 CPU 亲和性（affinity） 的特性，这意味着进程通常不会在处理器之间频繁迁移。这种状态正是我们希望的，因为进程迁移的频率小就意味着产生的负载小。

Linux Kernel 2.0之后，实现了CPU的硬亲和性，即：程序开发人员可以显式的指定进程在哪个或者哪些物理CPU上运行。

- Linux内核的硬亲和性

    在 Linux 内核中，所有的进程都有一个相关的数据结构，称为 task_struct。这个结构非常重要，原因有很多；其中与亲和性（affinity）相关度最高的是 cpus_allowed 位掩码。这个位掩码由 n 位组成，与系统中的 n 个逻辑处理器一一对应。 具有 4 个物理 CPU 的系统可以有 4 位。如果这些 CPU 都启用了超线程，那么这个系统就有一个 8 位的位掩码。每一位掩码对应系统中一个逻辑CPU。
    如果一个进程要使用某个 CPU，则该该进程的 task_struct 中的 cpu_allowed 属性中，对应于相应逻辑 CPU 的那一位掩码，应该置为 1， 反之，则置该位为 0， 这与网络中的网络掩码的概念比较类似。Linux 中，要想进程使用所有的逻辑 CPU，则需要将掩码的所有位均置为 1，这也是 Linux 进程的缺省状态。
    Linux 内核 API 提供了两个接口，可以让用户修改或者查看掩码，他们分别是：
        - sched_set_affinity()  修改进程掩码
        - sched_get_affinity()  查询进程掩码
    注意：sched_set_affinity()具有继承关系，即：会将cpus_allowd属性传递给子进程或者线程。

- KVM 使用硬亲和性

    通常 Linux 内核都可以很好地对进程进行调度，在应该运行的 CPU 上运行进程（这就是说，在可用的处理器上运行并获得很好的整体性能）。内核包含了一些用来检测 CPU 之间任务负载迁移的算法，可以启用进程迁移来降低繁忙的处理器的压力。
    一般情况下，在应用程序中只需使用缺省的调度器行为（Linux 软亲和性）。然而，您可能会希望在虚拟化中，使用 Linux 的硬亲和性来提高虚拟机的性能及稳定性。虚拟机用来做大数据处理，科学计算时，利用 Linux 硬亲和性指定了相应的逻辑 CPU 组时，可以减少进程在 CPU 之间的切换，提高缓存命中率，同时减少进程切换的时间，提高虚拟机的计算性能。又或者虚拟机是用来跑客户的关键业务时，需要一定的高效率和稳定性时，利用 QEMU 相关接口将虚拟机的 vCPU 绑定在物理 CPU 上。
    将虚拟机的 vCPU 绑定在物理 CPU 上：

    - 通过虚拟机xml文件的 vcpu 标签的 cupset 属性指定
        ```bash
        # 查看物理 CPU 架构
        cat /proc/cpuinfo
        virsh nodeinfo

        # 将虚拟机所有 vCPU 绑定在一组物理 CPU 上，编辑虚拟机的xml文件
        virsh edit instance_name
        <vcpu cpuset="1-4,^3,6">2</vcpu>

        # 查看 vCPU 与物理 CPU 之间的映射关系
        virsh vcpuinfo instance_name
        ```
        `<vcpu cpuset="1-4,^3,6">2</vcpu>` : 虚拟机一共两个 vCPU，将他们绑定在物理 CPU 的 1，2，4，6 号CPU上，在 vcpu 这个标签中，cpuset 属性是以整个虚拟机的所有 vCPU 为单位的，因此无法指定单个 vCPU与物理 CPU 之间的绑定关系。

    - 通过虚拟机xml文件的 cputune 标签来指定
        ```bash
        virsh edit instance_name
        ```
        ```xml
        <cputune>
            <vcpupin vcpu="0" cpuset="1"/>
            <vcpupin vcpu="1" cpuset="2"/>
        </cputune>
        ```
        将 vCPU 0 绑定到物理机的 CPU 1 上； 将 vCPU 1 绑定到物理 CPU 2 上。通过 cputune 标签，可以分别指定虚拟机的每个 vCPU 与物理 CPU 的绑定关系，控制更加精细，且优先级要高于 vcpu 标签中的 cpuset 属性。

    - 通过 `virsh` 命令设置
        ```bash
        # 将虚拟机 vCPU 0 绑定到物理 CPU 24 上
        virsh vcpupin instance_name 0 24
        ```
[参考](http://www.ibm.com/developerworks/cn/linux/l-affinity.html)

