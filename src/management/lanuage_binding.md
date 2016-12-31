# Libvirt Python Binding

libvirt 是用C语言实现的API，在简介中我们已经提到，libvirt为其他更高层的管理工具和应用程序提供统一的虚拟化功能操作接口，由于其功能的强大但是语言的局限，libvirt API被封装成支持多种语言的API。包括 c#, go, java, ocaml. perl, python, php, ruby。

## 示例
以OpenStack项目所用语言为例，本章讲述libvirt Python API。

在使用Python调用libvirt之前，我们需要安装libvirt-python软件包：

    [root@server-42.103.hatest.ustack.in ~ ]$ rpm -q libvirt-python
	libvirt-python-1.2.17-2.el7.x86_64

Python语言提供了到libvirt C语言函数的比较完整的绑定，绑定主要围绕`virConnect`和`virDomain`这两个对象。Python将libvirt的C语言函数映射为Connect对象或Domain对象中的方法。 映射的方式是从C语言函数名称中除去前缀“virConnect”和“virDomain”，并将留下的名称的第一个字母变为小写。

如C语言函数：
`int virConnectNumOfDomains(virConnectPtr conn)`转换为Python对象中的方法：

    virConnect::numOfDomains(self)

`int virDomainSetMaxMemory(virDomainPtr domain, unsigned long memory)`转换为：

    virDomain::setMaxMemory(self, memory)

只存在两个特例：
`virConnectListDomains`函数被替换为`virDomain::listDomainsID(self)`，它返回一个由域ID组成的列表，表示当前正在运行的域。
`virDomainGetInfo`函数被替换为`virDomain::info()`，它返回一个含有五个元素的列表，列表内容为：

1. state：状态值；
2. maxMemory：域的最大内存容量；
3. memory：域当前使用的内存量；
4. nbVirtCPU：vCPU的数量；
5. cpuTime：域使用的CPU时间（nanoseconds单位）。

以下是python调用python libvirt API的简单示例：

```bash
import libvirt

conn = libvirt.open('qemu+ssh://root@10.0.103.68/system')

for id in conn.listDomainsID():

    dom = conn.lookupByID(id)
    print "Dom %s  State %s" % ( dom.name(), dom.info()[0] )

    dom.suspend()
    print "Dom %s  State %s (after suspend)" % ( dom.name(), dom.info()[0] )

    dom.resume()
    print "Dom %s  State %s (after resume)" % ( dom.name(), dom.info()[0] )

    dom.destroy()
```

执行结果如下：

```bash
[root@server-42.103.hatest.ustack.in test ]$ python libvtest.py 
Dom instance-00000719  State 1
Dom instance-00000719  State 3 (after suspend)
Dom instance-00000719  State 1 (after resume)
```

## 参考文档

1. libvirt API的Python语言绑定：http://blog.sina.com.cn/s/blog_da4487c40102v367.html
2. Python API Binding：https://libvirt.org/python.html