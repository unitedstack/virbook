libvirt 是用C语言实现的API，在简介中我们已经提到，libvirt为其他更高层的管理工具和应用程序提供统一的虚拟化功能操作接口，由于其功能的强大但是语言的局限，libvirt API被封装成支持多种语言的API。包括 c#, go, java, ocaml. perl, python, php, ruby。

以OpenStack项目所用语言为例，本章讲述libvirt Python API。

在使用Python调用libvirt之前，我们需要安装libvirt-python软件包：

    [root@server-42.103.hatest.ustack.in ~ ]$ rpm -q libvirt-python
	libvirt-python-1.2.17-2.el7.x86_64

