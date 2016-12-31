# 中断与异常

## 中断架构

- 可编程中断控制器
- PIC
- APIC
- 处理器间中断
- 中断的重要概念

    - 中断分类

    外部中断：指连接在 IOAPIC 上设备产生的中断、LAPIC上连接的设备或者 LAPIC 内部中断源产生的中断以及处理器间的中断。

    可屏蔽中断：指可以通过某种方式（例如 CLI 命令，TPR）进行屏蔽的中断。与之对应的还有不可屏蔽中断。

    软件产生中断：指通过 INT n 指令产生的中断。

    - 中断的优先级

    - 中断的屏蔽

    CLI/STI 指令
    TPR 寄存器
    PIC/IOAPIC 的中断屏蔽位

    - IDT 表

    - 中断门

## 异常架构

和中断相比，异常最大的不同在于它是在程序的执行过程中同步发生的。根据产生的原因及严重程度，异常可以分为如下三类：

- 错误 (Fault)
- 陷阱 (Trap)
- 终止 (Abort)

## 操作系统对中断/异常的处理流程

虽然各种操作系统对中断/异常的处理实现不同，但都遵循如下的顺序：

- CPU 通过 vector 索引 IDT 表得到相应的“门”，获得其处理函数的入口地址。
- 程序跳转到处理函数执行，由于处理函数存放在 CPL=0 的代码段，程序可能会发生权限提升，处理函数通常执行下列几个步骤：

1. 保存被打断任务的上下文，并开始执行处理函数。

2. 如果是中断，处理完后需要些 EOI 寄存器应答，异常不需要。

3. 恢复被打断的任务的上下文，准备返回。

- 从中断/异常的处理函数返回，恢复被打断的任务，使其继续运行。

MSI (Message Signalled Interrupts) 中断方式。

## 参考文档

Interrupts And Exceptions：http://www.internals.com/articles/protmode/interrupts.htm

Message Signalled Interrupts：http://fundasbykrishna.blogspot.hk/2013/05/message-signaled-interrupt.html
