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