# 虚拟机资源管理

本章主要介绍Libvirt提供的用于管理和控制虚拟机资源占用和性能的方法。

使用libvirt来管理基于KVM的虚拟机时，libvirt会为每个虚拟机创建一个配置文件，用来保存虚拟机的配置信息和资源需求。其中有些信息是用来告诉KVM如何启动虚拟机，有些信息则是告诉虚拟机所在的宿主机，虚拟机需要哪些资源，可以对虚拟机的资源做什么限制。到libvirt 0.9.11为止，libvirt已经支持的多种资源控制，包括CPU、内存、块设备I/O、网络I/O。借助于cgroups，libvirt对CPU、内存、网络I/O的控制效果还是很理想，但对于块设备I/O的控制则相对较差。下面详细介绍每种资源控制的实现方法。

## 一、CPU

KVM在实现时，一个VCPU对应宿主机的一个线程，用于执行虚拟机操作系统的代码。除了需要一个与VCPU对应的线程，KVM还会为虚拟机的I/O操作启动一个或者多个线程，用来模拟I/O操作。因此，对KVM虚拟机的CPU资源控制时，需要将这两种类型的线程都考虑进去。

libvirt将CPU相关的信息配置在一个全局的cputune标签中，如下所示：

```bash
<domain>
	...
	<cputune>
		<vcpupin vcpu="0" cpuset="1-4,^2"/>
		<vcpupin vcpu="1" cpuset="0,1"/>
		<shares>2048</shares>
		<period>1000000</period>
		<quota>-1</quota>
		<emulatorpin cpuset="1-3"/>
		<emulator_period>1000000</emulator_period>
		<emulator_quota>-1</emulator_quota>
	</cputune>
	...
</domain>
```

上面配置中的关键字及其含义如下：

- vcpupin：用来指定每个vcpu线程可以运行的物理cpu的哪些核上。默认情况下，vcpu线程可以在物理CPU的任何一个核上运行。若指定，则必须指明vcpu编号和物理CPU核的编号。
- emulatorpin: 与vcpupin类似，用于指定非vcpu线程能够在物理CPU的哪些核上运行。
- shares：用于指定不同group的进程按照什么样的权重分享CPU，假如group1.shares是1024，group2.shares是2048，那么在出现资源竞争的情况下group2所得到的CPU资源就是group1的两倍。具体效果取决于是否出现资源竞争及不同group的比值。另外，在启用了quota之后，资源控制的效果主要取决于quota的设置。
- period：指定一个时间间隔，在该时间间隔内，某个虚拟机的运行时间不能超过quota设置的值。period的单位是毫秒（ms），范围：1000~1000000。
- quota：在period时间内，虚拟机所能运行的最长时间。单位是毫秒（ms），范围：1000~18446744073709551，负数表示不作限制。在多核的系统上，quota的值可以超过period，虚拟机所能达到的最大cpu利用率与vcpu的数目及quota/period的设置有关。
- emulator_period”：指定emulator在emulator_period的时间间隔内，最长的运行时间不能超
- emulator_quota：指定的时间长度。单位是毫秒（ms），范围：1000~1000000。
- emulator_quota：与quota类似，但用于限制emulator的运行时间。emulator主要用来模拟虚拟机的I/O操作。

上面的参数不仅仅可以限制虚拟机的CPU占用情况，也会对虚拟机的I/O造成影响。

virsh工具对虚拟机CPU的管理有多种方式，我们可以通过`virsh help | grep cpu`查看所有与cpu相关的内容。
以查看虚拟机的cpu信息为例：

```bash
[root@server-68.103.hatest.ustack.in libvirt ]$ virsh list
 Id    Name                           State
----------------------------------------------------
 8     instance-00000719              running

[root@server-68.103.hatest.ustack.in libvirt ]$ virsh vcpuinfo instance-00000719
VCPU:           0
CPU:            28
State:          running
CPU time:       25.6s
CPU Affinity:   ----------yyyyyyyyyyyyyyyyyyyyyyyyyyyyyy

VCPU:           1
CPU:            20
State:          running
CPU time:       15.3s
CPU Affinity:   ----------yyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
```

## 二、内存

与CPU类似，除了虚拟机的操作系统会消耗内存，运行于用户空间的emulator也会消耗内存，而这部分内存的大小不容易确定，经常被忽略掉，但是在用libvirt做内存限制时必须同时考虑到这两部分的内存。另外，由于linux内存管理上采用了写时复制技术，虚拟机实际的内存占用与分配给虚拟机的内存大小通常是不一样的。与CPU资源限制类似，做内存资源限制时，相应的参数都放在全局标签memtune中。如下：
```
<domain>
	...
	<memtune>
		<hard_limit unit='G'>1</hard_limit>
		<soft_limit unit='M'>128</soft_limit>
		<swap_hard_limit unit='G'>2</swap_hard_limit>
		<min_guarantee unit='bytes'>67108864</min_guarantee>
	</memtune>
	...
</domain>
```
上面配置中的关键字及其含义如下：单位都是（KB）

- hard_limit: 指定虚拟机最多能占用多上内存。
- soft_limit: 指定在出现内存竞争的情况下，虚拟机的内存限制。
- swap_hard_limit: 指定加上swap，虚拟机最多能占用的内存大小，必须比hard_limit大。
- min_guarantee: 指定最少需要给虚拟机分配的内存大小。

virsh内存相关命令：

```bash
[root@server-68.103.hatest.ustack.in libvirt ]$ virsh help | grep mem
    memtune                        Get or set memory parameters
    setmaxmem                      change maximum memory limit
    setmem                         change memory allocation
    dommemstat                     get memory statistics for a domain
    freecell                       NUMA free memory
    node-memory-tune               Get or set node memory parameters
    nodememstats                   Prints memory stats of the node.
```

## 三、块设备I/O

目前由于cgroups对于块设备的控制还比较有限，还不能实现细粒度的控制。所以libvirt也只能提供一些非常简单的控制。libvirt中实现的blkiotune是全局性的控制，只能按一定的权重来控制虚拟机对于物理设备的访问。如下：

```
<domain>
...
	<blkiotune>
		<weight>800</weight>
		<device>
			<path>/dev/sda</path>
			<weight>1000</weight>
		</device>
		<device>
			<path>/dev/sdb</path>
			<weight>500</weight>
		</device>
</blkiotune>
```

上面配置中的关键字及其含义如下：

- device: 每个device代表一个物理设备，path用于指定其绝对路径，weight用于指定虚拟机访问在该设备上的资源时的权重。

具体效果取决于是否出现竞争及各个虚拟机weight值的比例。

## 四、磁盘I/O

blkiotune是全局的设置，对于虚拟机的每个具体设备的I/O操作的控制则放在各个设备的iotune标签中。以磁盘设备为例，配置如下：
```
<devices>
	<disk type='file' snapshot='external'>
		<driver name="tap" type="aio" cache="default"/>
		<source file='/var/lib/xen/images/fv0' startupPolicy='optional'>
			<seclabel relabel='no'/>
		</source>
		<target dev='hda' bus='ide'/>
		<iotune>
			<total_bytes_sec>10000000</total_bytes_sec>
			<read_iops_sec>400000</read_iops_sec>
			<write_iops_sec>100000</write_iops_sec>
		</iotune>
	...
	</disk>
	...
<devices>
```

上面配置中指定了一个文件作为虚拟机的磁盘，并通过iotune标签对虚拟机的磁盘访问操作进行了限制,iotune标签中所有可用的参数及其含义如下：

- total_bytes_sec: 每秒最大的吞吐量。不能与read_bytes_sec和write_bytes_sec同时出现。
- read_bytes_sec: 每秒读操作的最大吞吐量。
- write_bytes_sec: 每秒写读操作的最大吞吐量。
- total_iops_sec: 每秒最大I/O操作次数。不能与read_iops_sec和write_iops_sec同时出现。
- read_iops_sec: 每秒最大读操作次数。
- write_iops_sec: 每秒最大写操作次数。
- 
libvirt是通过的Qemu的I/O throttling来实现本功能的。


## 五、网络I/O
与磁盘I/O类似，网络I/O也是每个设备单独设置的，配置在设备的bandwidth标签中。如下：
```
<devices>
	<interface type='network'>
		<source network='default'/>
		<target dev='vnet0'/>
		<bandwidth>
			<inbound average='1000' peak='5000' floor='200' burst='1024'/>
			<outbound average='128' peak='256' burst='256'/>
		</bandwidth>
	</interface>
</devices>
```
上面配置中的关键字及其含义如下：单位（KB）

- iobound: 设置数据包进入虚拟机的最大速率。其中average是必不可少的，floor是最低速率，peak是峰值，burst则是在峰值速度是能传输的最大数据量。
- outbound: 设置虚拟机发送数据包的最大速率。其中average是必不可少的，peak是峰值，burst则是在峰值速度是能传输的最大数据量。
- bandwidth标签中最多只能有一个inbound和一个outbound，可以只指定输入或输出带宽。