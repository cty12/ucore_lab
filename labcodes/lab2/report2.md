# Lab2 Report

## Exercise 00

> 将 lab1 的工作移植到 lab2 的初始代码。

使用 meld 完成了半自动移植。

## Exercise 01

> 实现 first fit 连续内存分配算法，需要修改文件 default_pmm.c 。  
> 注意在链表中需要按空闲块起始地址排序形成有序的链表。

实现上参考了 result ，一个稍优的算法会在“改进空间”部分给出。主要修改 `default_pmm.c` 文件中的三个函数，分别是：初始化用的 `default_init_memap` ，分配页使用的 `default_alloc_pages` ，释放页用的 `default_free_pages` 。  

初始化函数的修改比较简单，只需要将当前的以 `base` 作为基址长度为 `n` 的页按顺序加入链表，并设定好 Page 的 `property` 值就可以了，除了 base 之外其他页的 property 都为 0 。其他部分不用修改，和初始代码完全相同。  

分配页的时候首先要检查一下 n （就是要分配的大小）的值是不是过大，如果比当前所有空闲页的大小还要大那么肯定是分配失败的。遍历 `free_list` 得到地址最小的 property 大于等于 n 的页。如果遍历不能得到这样的页，也是分配失败。逐一将该页后面的页面移出 `free_list` 并 set reserved 。最后如果后面还有空闲页面的话，将 n 个页面之后的 list entry 的 property 设置为原结点的 property - n （因为已经分配出去 n 个页面了）。如果没有找到合适的块，分配失败。

释放页的时候一开始也很简单，主要难点在相邻的空闲块的合并。释放的时候只需要在 `free_list` 里找出第一个地址大于等于 `base + n` 的（因为链表是按照地址顺序储存的，不过因为地址是连续的，这里写 `p > base`也是对的），然后把要释放的 n 个页面直接顺序插入进 `free_list` 就可以了。然后把 `base` 的引用位置 0 ， property 置为 n （因为释放后后面有了 n 个可用的页面）。然后就是合并新加入的空闲块和它相邻的空闲块，合并的时候，因为这些块里面的页面都已经位于 `free_list` 中了，所以只需要调整它们最前面（就是地址最低）的块的 property 就可以了。然后就完成了空闲块的合并。

first fit 内存分配算法就此完成了。

> 是否有改进空间？

显然有改进空间。现在在链表 `free_list` 储存所有 free 的页，这是不经济的，因为很多 property 域为 0 的页根本没有用，在遍历链表的时候都把它们忽略了。那么，`free_list` 事实上可以只储存 free 区域的头一个 Page ，alloc 的时候只需要在 `free_list` 里找一个 property >= n 的 list_entry 出 list 然后把代表剩余空间的 `list_entry` 放回 list 就可以了。这样可以显著减少遍历链表的时间。

不过这种方式在 alloc 的时候还是得遍历 n 个然后 SetPageProperty ，free 的时候快不少。

## Exercise 02

> 实现寻找虚拟地址对应的页表项。  

主要修改 get_pte 函数。寻找虚地址对应的二级页表项的内核虚地址的方法是：

```
return & ((pte_t *) KADDR(PDE_ADDR(* pdep)))[PTX(la)]
```

如果页表项不存在，那么需要给页表分配一个页，然后设置引用，使用 `memset` 清空它的内容就可以了。

> PDE 和 PTE 的每个组成部分的含义和对 ucore 的用途。

PTE 每个部分的组成如下：

```
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
                                                // hardware, so user processes are allowed to set them arbitrarily.

```

> ucore 执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

Page Fault 首先会发出一个异常，操作系统通过 IDT 得出这是一个缺页异常，跳转到异常处理代码。硬件处理的方式是使用替换算法替换掉较旧的一个页面，然后在主存中载入要访问的页面。

## Exercise 03

> 释放某虚地址所对应的页并取消对应二级页表项的映射。

和上面一题一样，完全按照代码里的注释的提示就可以了。如果页表项存在，减少页的引用计数，如果计数为 0 ，就释放该页面。取消二级页表项的映射，然后 flush TLB 。

> 数据结构 Page 的全局变量（数组）的每一项与页表的页表项与页目录项有无对应关系？若有，其对应关系是什么？

pages 数组里的就是页目录表和页表指向的物理页。

> 若希望虚拟地址与物理地址相等，则需要如何修改 lab2 ？

将 ucore 的起始地址和内核虚地址设为相同的值。
