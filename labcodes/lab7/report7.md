# Lab 7 Report

计 24 陈天昱 2012011348

## Exercise 0

使用 meld 完成移植。根据提示，对之前的代码进行修改。在 trap.c 中的 trap\_dispatch() 直接调用 run\_timer\_list() 替代 lab 6 中的 sched\_class\_proc\_tick() 。将后者的定义恢复为 static 。

## Exercise 1

> 内核级信号量的设计描述和执行流程。

看一眼 ./kern/sync/sem.h 就知道信号量的基本实现包括一个结构体和四个方法。

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

