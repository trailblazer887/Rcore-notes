#### 多道程序放置
通过`build.py`对不同的应用将`linker.ld`的`BASE_ADDRESS`作相应改动，从而实现将不同的ELF文件链接到不同的基地址，
#### 多道程序加载
通过`loader.rs`中的`load_app()`对不同应用进行加载
#### 执行应用程序
与`ch2`大体相同，不同的是：
1. 跳转到下一条应用程序(编号 $i$ )的的入口点
2. 相应的栈指针
---
#### 任务的概念
**任务**：单个应用程序一次完整的执行过程
**计算任务片**：一个时间片段上的执行片段
**空闲任务片**：一个时间片段上的空闲片段
**任务切换**：从一个程序的任务切换为另一个程序的任务
**任务上下文**：需要保存和恢复的资源
#### 任务上下文切换
<font color="yellow">本质</font>：来自两个不同的应用在内核中的**Trap控制流**的切换
主要函数：`__switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext)` --- 是`switch.S`中的`__switch`的封装。封装的目的是让**编译器自动完成调用者寄存器的保存**
切换过程( 从应用 A 到应用 B )：
1. 将A当前的被调用者寄存器(`ra`, `sp`(内核栈指针), `s0~s11`)保存在A的任务上下文空间里(`current_task_cx_ptr`指向的位置)
2. 根据B的任务上下文空间保存的内容(`current_task_cx_ptr`指向的位置)，恢复寄存器
3. `ret`返回到当前`ra`保存的地址(返回B的`__restore`处)
---
#### 协作式调度
通过一个`TaskManager` 的全局实例 `TASK_MANAGER`
实现`sys_yield`和`sys_exit`系统调用
`sys_yield`调用链( `sys_exit`同理 )：`sys_yield` -> `suspend_current_and_run_next` -> `mark_current_suspended 和run_next_task` -> `TASK_MANAGER.mark_current_suspended 和 TASK_MANAGER.run_next_task`

---
#### 抢占式调度
背景：由于应用向"批处理"转向“交互式”，应用的设计者不会轻易的放弃掉CPU的使用权(可能导致无法忍受的延迟)，所以OS收回(部分)调度的权力，将CPU资源合理分配给程序。调度算法需要考虑**性能(吞吐量和延迟)** 和 **公平性**。
使用的调度算法：**时间片轮转调度**(Round-Robin)
#### RISC-V架构中的中断
考虑S特权级下的中断(在S态下执行时发生的S态中断)：
1. `sstatus.sie`设置为1，且`sie`(它的三个字段`ssie/stie/seie`分别控制S特权级的软件中断，时钟中断，外部中断的中断使能)相应字段为1，该中断才不会被屏蔽
2. `sstatus.sie`设置为0，此时S态中断均被屏蔽 

<font color="red">注意</font>：高特权级的中断**可以打断**低特权级的执行，无需`sstatus.sie`和`sie`
> "当一个硬件线程运行在特权模式 `x` 时，中断的全局使能状态由 `xIE` 决定。而对于特权级比 `x` 更高的模式 `y`（即 `y>x`），其中断总是全局使能的，**不受**模式 `y` 自己的 `yIE` 位影响。"

- tips: 为了简单起见，内核不会在S态被S态中断所打断(即S态时`sstatus.sie`总为`0`)

中断产生后，硬件会做出以下举动：
1. 将`sstatus.sie`字段保存到`sstatus.spie`中，同时将`sstatus.sie`清零
2. 软件中断处理完毕后，执行`sret`指令返回到被打断的地方继续执行，硬件将`sstatus.sie`字段恢复为`sstatus.spie`字段中的值