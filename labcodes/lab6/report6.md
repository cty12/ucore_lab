# Lab 5 Report

计 24 陈天昱 2012011348

## Exercise 0

使用 meld 完成移植。

需要修改之前实验的几个地方。首先 alloc\_proc() 要初始化几个新的域；然后在 trap.c 里修改时钟中断的处理部分，然后修改 sched.c / sched.h 将 sched\_class\_proc\_tick() 前面的 void 删去，因为在 trap.c 里要使用。

## Exercise 1

> 请理解并分析 sched_class 中各个函数指针的用法，并结合 RR 调度算法描述 ucore 的调度执行过程。

使用的是 RR 算法，所以都是使用的 RR 实现的调度函数。

RR\_init() 用于初始化 run\_queue ；RR\_enqueue() 用于向 run_queue 中添加一个就绪态的进程；RR\_dequeue() 用于从 run\_queue 中移除一个进程；RR\_pick\_next() 用于从 run\_queue 中选取一个进程运行；proc\_tick() 作用就是在每次时钟中断到来时将当前进程的 time\_slice 减一。

时钟中断的时候直接调用 sche\_class\_proc\_tick 调整 current 进程的 time\_slice 。在 do\_exit() ，do\_wait() ，init\_main() ，cpu\_idle() 中都会调用 schedule() ，schedule() 干的事是将 current 进程放入 run\_queue ，并从 run\_queue 中选出 next 进程，并使用 proc\_run() 运行 current 。

这就是 ucore 的基本调度过程。

> 请在实验报告中说明如何设计实现多级反馈队列调度算法。

1. 维护多条不同优先级的 run\_queue 。新进程加入最顶层的 queue 尾部；
2. 队列头部的进程分配 CPU 运行；
3. 若进程在时间片用完之前退出，那么移出队列；
4. 若进程主动放弃 CPU ，移出队列。当进程再次就绪，放回到离开时的队列的队尾；
5. 若一个进程用完了时间片，它的优先级降低。将其插入低一级的队列的队尾；
6. 在最低级，进程按照 RR 算法调度，直至退出离开队列。

（参考资料：![Multilevel feedback queue](http://en.wikipedia.org/wiki/Multilevel_feedback_queue)）

## Exercise 2

> 实现 Stride 算法。首先用 default_sched_stride_c 覆盖 default_sched.c 然后在此文件的基础上实现 stride 算法。

实现以下几个方法：

- init: 初始化调度器类的信息，初始化 run\_queue ；
- enqueue: 初始化刚进入 run\_queue 的进程 proc 的 stride 域，将 proc 插入 run\_queue；
- dequeue: 从 run\_queue 中删除相应的元素；
- pick\_next: 返回 run\_queue 中 stride 最小的进程，更新对应进程的 stride 
- proc\_tick: 减小 time_slice 域。检测当前进程是否用完了分配的时间片，如果时间片用完，则需要设置标记引起进程切换。

## 小结

和参考答案的主要区别是初始化的时候使用 list\_init 初始化进程的 run\_link ，因为它给出的 RR 算法有一个检查，如果像答案那样初始化，assert(list\_empty(&(proc->run\_link))); 这个检查点肯定是过不去的。另外答案写了一个用 skew heap 的版本和一个不用 skew heap 的版本，而我只写了用 skew heap 的，因为既然定义好了这个数据结构为什么不用呢。
