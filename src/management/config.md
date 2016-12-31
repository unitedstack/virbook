# libvirt 配置

libvirt配置文件在`/etc/libvirt/`目录下，其一层目录树如下：

```
[root@server-68.103.hatest.ustack.in libvirt ]$ tree -L 1
.
├── libvirt.conf
├── libvirtd.conf
├── nwfilter
├── qemu
├── qemu.conf
├── qemu-lockd.conf
├── secrets
└── virtlockd.conf
```

**启动libvirt进程**

```bash
/usr/sbin/libvirtd --listen
```

下面将简单介绍部分重要的配置文件。

**（1）libvirt.conf**

**需要注意的参数**
1. `uri_aliases`：配置一些常用libvirt连接的别名。通过virsh调用这个别名与远程主机的libvirt连接。
```
uri_aliases = [
  "server-69=qemu+ssh://root@10.0.103.69/system",
]
```

操作如下：

    [root@server-68.103.hatest.ustack.in libvirt ]$ systemctl restart libvirtd
    [root@server-68.103.hatest.ustack.in libvirt ]$ virsh -c server-69
    Welcome to virsh, the virtualization interactive terminal.
    
    Type:  'help' for help with commands
       'quit' to quit
    
    virsh # 

**（2）libvirtd.conf**

该文件是libvirtd守护进程的配置文件。

**需要注意的参数**
1. `listen_tls = 0`：关闭TLS安全认证的连接。
2. `listen_tcp = 1`：打开TCP连接。
3. `tcp_port = "16509"`：配置TCP监听端口。
4. `listen_addr = "0.0.0.0"`：设置监听地址。
5. `auth_tcp = "none"`：无需密码或认证就可以访问，但是存在安全隐患。

**（3）qemu.conf**
该配置文件是libvirt对QEMU驱动的配置文件，包括 VNC、SPICE等和连接它们时采用的权限认证方式的配置，也包括内存大页、SELinux、Cgroups等相关配置。

**（4）qemu目录**
该目录下存放的是QEMU驱动的域，也就是虚拟机的xml文件。

```
.
├── autostart
├── instance-000006a8.xml
├── instance-00000717.xml
├── instance-00000719.xml
├── instance-0000071a.xml
├── instance-0000071c.xml
├── instance-0000071d.xml
├── instance-0000071e.xml
└── networks
    └── autostart
```

## 参考文档

1. LIBVIRT、LIBVIRTD的配置和使用：http://smilejay.com/2013/03/libvirt-configuration-and-usage/