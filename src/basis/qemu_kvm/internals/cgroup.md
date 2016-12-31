Linux 内核 2.6.24 提供了一个新的内核特性 —— control groups，简写为 cgroups。Cgroups 主要使用来提供对系统上任务组（进程）之间的资源，例如：CPU 时间，系统内存，网络带宽等的分配、统计以及隔离的功能。

这里推荐两个用于理解cgroups的参考链接：

[cgroups wiki](https://en.wikipedia.org/wiki/Cgroups)

[Introduction to Linux Control Groups](https://sysadmincasts.com/episodes/14-introduction-to-linux-control-groups-cgroups)

wiki中很好阐述了cgroups的历史、功能等基础概念问题；而后一篇是一个操作 Linux cgroups 的简单教程。

libvirt 使用 Linux cgroups 功能参考[Control Groups Resource Management](https://libvirt.org/cgroups.html)，它简单解释了 cgroups 在 libvirt 虚拟机中使用。
