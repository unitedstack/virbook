### 三种基本模式

- 实模式 (Real Mode)
- 保护模式 (Protect Mode)
- 虚拟 8086 模式 (Virtual 8086 Mode)

### 基本寄存器组

包括：

- 通用寄存器： 8 个 32 位。
- 内存管理寄存器
- EFLAGS 寄存器
- EIP 寄存器
- 浮点运算寄存器
- 控制寄存器
- 其他寄存器

### 权限控制

x86 提供两种权限控制机制——段保护与页保护

- 段保护

当前权限级别 (Current Privilege Level, CPL)表示当前运行的代码的权限。Ring0 对应于 CPL=0，具有最高权限，操作系统内核运行在该权限下；Ring3 对应于 CPL=3，用户程序运行在 Ring3，权限最低。

除此之外还有描述符权限级别 (Descriptor Privilege Level, DPL)与所要求权限级别 (Reqested Privilege Level, RPL)。

- 页保护

页保护的思想比段保护简单，它通过在页目录项、页表项中引入一个 User/Supervisor 位，将页面分成 User 和 Supervisor 两个特权级。Supervisor 模式对应 CPL=0，1，2；User 对应于 CPL=3。

### 参考文档

CPU Rings, Privileges, and Protection: http://duartes.org/gustavo/blog/post/cpu-rings-privilege-and-protection/
