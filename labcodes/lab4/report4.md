# Lab 4 Report

## Exercise 0

使用 meld 完成半自动移植。

## Exercise 1

### 实验步骤

这个练习的主要任务是初始化 PCB 内部的各数据段。

### 思考题

> 请说明 struct context 和 struct trapframe 成员变量的含义与作用。

context 储存的是进程的上下文，即各个寄存器的值：

```
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

trapframe 储存的是中断发生前栈帧的一些信息：

```
struct trapframe {
    struct pushregs tf_regs;
    uint16_t tf_gs;
    uint16_t tf_padding0;
    uint16_t tf_fs;
    uint16_t tf_padding1;
    uint16_t tf_es;
    uint16_t tf_padding2;
    uint16_t tf_ds;
    uint16_t tf_padding3;
    uint32_t tf_trapno;
    /* below here defined by x86 hardware */
    uint32_t tf_err;
    uintptr_t tf_eip;
    uint16_t tf_cs;
    uint16_t tf_padding4;
    uint32_t tf_eflags;
    /* below here only when crossing rings, such as from user to kernel */
    uintptr_t tf_esp;
    uint16_t tf_ss;
    uint16_t tf_padding5;
} __attribute__((packed));
```

## Exercise 2

### 实验步骤

填写 do\_fork() 函数。首先使用 alloc\_proc() 分配并初始化一个新进程，并将其 parrent 设为当前进程。然后为进程分配栈，然后复制原有进程的执行状态。然后将 PCB 加入 hash 表和 proc\_list ，最后将新进程设置为 runable 。

实验中需要注意使用 local\_intr\_save 和 local\_intr\_restore 打开和关闭中断，具体方法可以参考代码中的 proc_run() 函数。

### 思考题

> 是否能做到给每个新 fork 的线程一个唯一的 id ？说明理由。

get_pid() 函数可以保证每次分配不同的 PID 。get\_pid 将 pid 每次加一直到找到可用的 PID 返回。如果进程数等于 MAX\_PROCESS 从而没有 PID 可以分配，会不断进入 repeat 直到其中某个进程退出。从而 fork 出来的每个进程的 PID 都是唯一的。

## Exercise 3

### 思考题

> 分析 proc_run() 并回答以下问题：  

> 创建运行了几个内核线程？

两个，分别是 idle 和 init 。

> local_intr_save 和 local_intr_restore 在这里有何作用？

可以参考 ./kern/sync/sync.h 里的代码，local\_intr\_save 用于关闭 interrupt request ，local\_intr\_restore 用于恢复 interrupt request 。
