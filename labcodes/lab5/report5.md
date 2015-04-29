# Lab 5 Report

计 24 陈天昱 2012011348

## Exercise 0

使用 meld 完成半自动移植。

## Exercise 1

### 实验步骤

> 加载应用程序并执行

首先要更新前几次实验的代码，例如在设置 IDT 的时候加入主管系统调用的条目，将 proc_struct 里几个负责进程调度的域进行初始化等。

然后就是填写 load_icode 的代码。做的工作就是设置 trapframe ，将段寄存器设置为用户段的相应的值，栈顶寄存器 ESP 设置为用户栈的栈顶，指令寄存器 EIP 设置为程序的入口地址，EFLAGS 设置为允许中断。

### 思考题

> “创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的”

通过阅读代码可以知道在 load\_icode() 中加载用户程序并设置了 trapframe 。进行 SYS\_exec 系统调用之后，使用 iret 指令返回继续执行 trapframe 中 tf\_eip 指向的地址，而这里就是用户程序的入口地址，从而用户程序就执行起来了。

## Exercise 2

### 实验步骤

这个实验填写 copy_range() 函数的内容，作用是将进程 A 的内存内容拷贝到进程 B 。首先获得源页和目标页的内核虚地址，然后使用 memcpy 将这页复制过去，最后设置与物理地址的对应关系。

### 思考题

> 如何实现 COW 机制？

可以增加一个标记位，比如叫做 cow\_flag 。copy\_range 的时候不复制页，而只是修改这个标记位为 1 ，当进程要写入该页的时候，如果标记位为 1 ，则再通过 trap 进入处理程序，使用 memcpy 复制内存页。

### Exercise 3

> “请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？”

- 通过 fork() 新建的进程起初状态为 UNINIT ，通过 wakeup_proc() 变为 RUNNABLE ；

- exec 时，进程一直为 RUNNING 状态；

- wait 时，若存在子进程则变为 SLEEPING 状态；

- exit ，变为 ZOMBIE 状态。

> 用户态进程的生命周期图。

```

process state changing:
                                            
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+ 
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  + 
                                           -----------------------wakeup_proc----------------------------------
```

### 小结

这个实验完全按照注释中的实验提示完成即可，和参考解答只有细节上的差异。
