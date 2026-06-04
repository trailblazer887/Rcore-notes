### trap_handler函数的调用链
以`trap_handler`处理`write`系统调用为例 (**从S到M**)
`trap_handler` -> `syscall` -> `sys_write` -> `print!`(属于console.rs) -> `print` -> `write_fmt` -> `write_str`(属于Write trait) -> `console_putchar`(sbi_call的封装接口) -> `sbi_call`
### CSR概述
|   CSR 名   |            该 CSR 与 Trap 相关的功能             |
| :-------: | :---------------------------------------: |
| `sstatus` | `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
|  `sepc`   | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址  |
| `scause`  |                描述 Trap 的原因                |
|  `stval`  |               给出 Trap 附加信息                |
|  `stvec`  |             控制 Trap 处理代码的入口地址             |
触发Trap时，主要是将**32个通用寄存器**和**CSR寄存器**`sstatus`和`sepc`推入内存栈(`scause/stval`**总是在 Trap 处理的第一时间**就被使用或者是在其他地方保存下来了，因此它没有被修改并造成不良影响的风险，而`sstatus/sepc`在Trap全程都有意义，有时还会因Trap嵌套而覆盖掉)
### 简要流程
`__alltraps`(保存寄存器到内核栈) -> `trap_handler`(处理中断/异常) -> `__store`(弹出内核栈的内容到寄存器)

### 课程作业
<font color="red">注意：</font>课后练习的后**4个编程题不做**！这些都是有关后面章节内容的

### 实验练习
#### 问答作业
1. 我所使用的sbi版本：`[rustsbi] RustSBI version 0.3.0-alpha.4, adapting to RISC-V SBI v1.0.0`
2. 问题2：
	1. 刚进入`__restore`时，`a0`代表**内核栈的栈指针**。`__restore`的使用场景：(1) 当调用`run_next_app`的时候，重置`sepc`寄存器为`APP::BASE::ADDRESS`，`sstatus`为`U`，`sp`最后存放用户栈指针，`sscratch`最后存放内核栈栈指针，为执行初始/下一个`app`做准备。 (2) 当用户态触发中断或陷阱时，`__restore`用于恢复触发前的上下文(寄存器等)，保证处理完异常后用户态程序能够正确运行。
	2. 见以上笔记
	3. `x2`在之前已经转移进`sscratch`里，而`x4`一般来说用不到
	4. `sp`中为用户态栈指针，`sscratch`中为内核态栈指针
	5. `sret`后发生状态切换。因为`sstatus`的`spp`字段给出了进入`trap`之前的特权级`U`
	6. `sp`中为内核态栈指针，`sscratch`中为用户态栈指针
	7. `ecall`
	8. 并不是。可以按照异常的类型动态判断保存哪些寄存器