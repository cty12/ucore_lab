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

结构体包含两个部分一个是 value 值，另一个是等待队列。

```c
void
sem_init(semaphore_t *sem, int value) {
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}
```

sem_init() 初始化了 value 值和等待队列。

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```

up() 方法调用了 \_\_up 。首先关闭中断。如果队列为空就将 value + 1 ；否则唤醒等待队列队首的进程。最后恢复中断。

```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

down() 方法调用了 \_\_down 。首先关闭中断。如果 value > 0 ，则 value - 1 直接返回恢复中断；否则将当前进程加入等待队列，恢复中断。然后调用 schedule() 交出 CPU 使用权，再次关闭中断，从等待队列中删除当前进程，最后重新恢复中断。

> 给出给用户态进程/线程提供信号量机制的设计方案，并比较和给内核级提供的异同。

实现原理上没有区别，只是需要将相关的方法设置成系统调用的形式，这样才能在用户态直接调用。

## Exercise 2

> 给出内核级条件变量的设计描述并说明大致执行流程。

条件变量是在 condvar 里定义的：

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

需要用到的是 monitor 结构体：

```c
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

相关的方法在 monitor.c 中实现，定义如下：

```c
// Initialize variables in monitor.
void     monitor_init (monitor_t *cvp, size_t num_cv);
// Unlock one of threads waiting on the condition variable.
void     cond_signal (condvar_t *cvp);
// Suspend calling thread on a condition variable waiting for condition atomically unlock mutex in monitor,
// and suspends calling thread on conditional variable after waking up locks mutex.
void     cond_wait (condvar_t *cvp);
```

init\_monitor() 用于初始化 monitor 结构体；cond\_signal() 用于唤醒一个在条件变量上等待的进程；cond\_wait() 表示该进程等待某个条件需要睡眠。

具体实现上，可以参考实验指导书上的伪代码：

```
if(cv.count > 0) {
	monitor.next_count ++;
	sem_signal(cv.sem);
	sem_wait(monitor.next);
	monitor.next_count --;
}
```

```
cv.count ++;
if(monitor.next_count > 0)
	sem_signal(monitor.next);
else
	sem_signal(monitor.mutex);
sem_wait(cv.sem);
cv.count --;	
```

cond\_signal() 和 cond\_wait() 完全按照上述实现就可以了。然后使用条件变量实现哲学家就餐问题的关键函数 phi\_take\_forks\_condvar() 和 phi\_put\_forks\_condvar() 就可以完成对信号量部分的测试。

> 给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明与给内核级提供条件变量机制的异同。

## 小结

都是按照信号量实现的，和答案的方法没有明显区别。
