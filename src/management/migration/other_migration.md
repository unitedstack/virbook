### 一、数据传输方式

从上一篇中迁移的实现过程我们可以看到，虚拟机的迁移其实就是数据的迁移，从这个角度来看，libvirt中数据的传送方式可以分为如下两种：

- Native方式：利用虚拟机本身在两个宿主机之间迁移
- Tunnel方式：利用libvirtd后台进程迁移
#### Native方式

Hypervisor原生传输数据的方式一般不支持数据的加密，该功能是否支持依赖于Hypervisor本身。但是这种传输方式消耗的资源少。
使用这种方式传输数据的话需要管理员在部署主机时额外配置好hypervisor的网络。若要支持并发迁移，需要使用到多个网络端口，这样的情况就需要iptables提前开启这些tcp的端口。

![](/images/management/native_migration.png)

#### Tunnel方式
虚拟机的迁移主要由第三方libvirt来控制，通过libvirt的RPC调用，实现远程传输，且支持数据加密。

但是这种方式存在其弊端：在源主机和目的主机中都会存在该虚拟机的副本，因为数据是在libvirt和Hypervisor之间传递。对于RAM很大的虚拟机来说，这个弊端会导致很快产生脏数据。
在部署方面，通过隧道传输不需要做额外的网络配置，只需要在防火墙上打开一个端口就可以支持多个并发操作。
上一篇章中的在线迁移方式就是隧道传输的典型应用。

![](/images/management/tunnel_migration.png)


### 二、数据管理方式

虚拟机的迁移需要源主机、目的主机和第三方主机（如果有的话）之间，以及迁移需要调用的应用程序之间的密切协调合作。该过程中有两种数据管理方式：

- Direct迁移
- Peer to Peer迁移

#### Direct迁移

这种模式下，由libvirt客户端进程来控制整个迁移过程，因此，libvirt客户端需要与源主机和目的主机之间相连并得到它们的授权。这样的话，源主机和目的主机上的libvirtd进程就不需要相互通信。如果迁移过程中libvirt客户端崩溃，或者丢失了与libvirtd之间的连接，则libvirt客户端将尝试终止迁移过程并在原主机上重新启动虚拟机。若该操作无法安全的执行，所有主机上的虚拟机都将被暂停。

![](/images/management/direct_managed.png)

#### Peer to Peer迁移

这种模式下，libvirt客户端只与源主机的libvirtd守护进程通信，然后源主机的libvirtd通过直接连接目的主机的libvirtd守护进程来控制整个迁移过程。和Direct不同的是，如果出现迁移过程中libvirt客户端崩溃，或者丢失了与源主机libvirtd之间的连接，迁移过程依然会不间断的继续执行，直到迁移完成。需要注意的是，源主机认证（通常是root）链接到目的主机，而不是通过客户端应用链接到源主机。

![](/images/management/peer_to_peer.png)

QEMU Driver支持如下几种迁移方式：

1. Native migration, client to two libvirtd servers

```bash
用法：virsh migrate GUESTNAME DEST-LIBVIRT-URI [HV-URI]

使用默认的网络接口
[root@server-68.103.hatest.ustack.in ~ ]$ virsh migrate instance-0000071e qemu+ssh://10.0.103.69/system
[root@server-68.103.hatest.ustack.in ~ ]$ virsh migrate instance-0000071e qemu+tls://10.0.103.69/system

使用新建的网络接口
[root@server-68.103.hatest.ustack.in ~ ]$ virsh migrate instance-0000071e qemu://10.0.103.69/system tcp://10.0.103.69/
```

2. Native migration, client and peer2peer between, two libvirtd servers
	
此模式下不能使用virsh

3. Tunnelled migration, client and peer2peer between two libvirtd servers 

此模式下不能使用virsh

4. Native migration, peer2peer between two libvirtd servers 

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh migrate --p2p instance-0000071d qemu+ssh://10.0.103.69/system
```
5. Tunnelled migration, peer2peer between two libvirtd servers

```bash
[root@server-68.103.hatest.ustack.in ~ ]$ virsh migrate --p2p --tunnelled instance-0000071e qemu+ssh://10.0.103.69/system
```


### 参考资料
1.	[https://libvirt.org/migration.html](https://libvirt.org/migration.html)
