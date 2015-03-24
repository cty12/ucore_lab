# Lab1 Report

## Assignment 01

> 操作系统镜像文件 ucore.img 是如何一步一步生成的？

采用静态分析代码和通过 `make "V="` 查看调试信息结合的办法。

可以看到最后几步的调试信息如下：

```
dd if=/dev/zero of=bin/ucore.img count=10000
记录了10000+0 的读入
记录了10000+0 的写出
5120000字节(5.1 MB)已复制，0.0432516 秒，118 MB/秒
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
记录了1+0 的读入
记录了1+0 的写出
512字节(512 B)已复制，0.000262411 秒，2.0 MB/秒
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
记录了138+1 的读入
记录了138+1 的写出
70775字节(71 kB)已复制，0.00242387 秒，29.2 MB/秒
```

我们知道 dd 的默认 block 大小是 512B ，首先创建了一个 512 * 10000B 的全零的文件 ucore.img，
然后将 bin/bootblock 和 bin/kernel 复制进来形成镜像文件。

再分析一下 bootblock 和 kernel 是如何得到的。
生成 bootblock 的相关代码为：

```
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

主要需要的是 bootasm.o 和 bootmain.o 和 sign。

通过 bootasm.S 生成 bootasm.o 的方法为：

```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
```

类似的，通过 bootmain.c 生成 bootmain.o 的方法为：

```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
```

关键参数：

|   参数      |                   含义                           |
|------------|--------------------------------------------------|
| -fno-builtin |    除非用 __builtin__ 前缀否则不进行 builtin 函数的优化；|
| -Wall        |    warning all  |
| -ggdb        |    生成 gdb 使用的调试信息  |
| -m32         |    生成适用于 32 bit 环境的代码 |
| -gstabs      |    生成 stabs 格式的调试信息，这样可以显示出调用栈信息 |
| -nostdinc    |    不使用标准库              |
| -fno_stack-protector | 不生成检测缓冲区溢出的代码 |
| -Os          |    为减小代码大小进行优化     |
| -I <dir>     |    添加搜索头文件的路径       |

生成 sign 工具的方法为：

```
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```

通过 bootasm.o 和 bootmain 链接生成 bootblock.o ：

```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```

关键参数：

|    参数     |        含义        |
| -m elf_i386 | 模拟为 i386 上的连接器 |
| -nostdlib   | 不使用标准库   |
| -N          | 设置代码段和数据段均可读写 |
| -e <entry>  | 指定入口  |
| -Ttext      | 指定代码开始段位置 |

将 bootblock.o 拷贝到 bootblock.out 。

使用 sign 处理 obj/bootblock.out 生成 bin/bootblock 。

生成 kernel 的相关代码为：

首先需要 kernel.ld init.o readline.o stdio.o kdebug.o kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o trapentry.o vectors.o pmm.o  printfmt.o string.o 这几个文件。

这些 \*.o 的生成方法都类似。

然后生成 kernel ：

```
ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
obj/kern/init/init.o obj/kern/libs/readline.o \
obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
obj/kern/driver/clock.o obj/kern/driver/console.o \
obj/kern/driver/intr.o obj/kern/driver/picirq.o \
obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
obj/libs/printfmt.o obj/libs/string.o
```

-T 为指定链接使用的脚本。

> 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

磁盘的主引导扇区为 512 byte ，且倒数第二个是 0x55 ，最后一个是 0xAA 。

## Assignment 02

> 使用 qemu 执行并调试 lab1 中的软件。  
> 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。  

修改 tools/gdbinit 为：

```
set architecture i8086
target remote :1234
```

然后使用 `make debug` 进入调试模式，通过 `layout prev` 改变显示的 layout 来显示寄存器和反汇编得到的汇编指令。

通过 `si` 单步执行。

> 在初始化位置 0x7c00 设置实地址断点，测试断点正常。  

设置断点 `br *0x7c00` ，然后使用 `info breakpoints` 查看设置的断点。`continue` 运行，运行正常。

> 从 0x7c00 开始跟踪代码运行，将单步跟踪反汇编得到的代码与 bootasm.S 和 bootblock.asm 进行比较。

还是与之前类似的方法在 0x7c00 设置断点，使用 `x /10i $pc` 查看当前 PC 寄存器指向的地址及之后的 10 条指令。

得到的结果是：

```
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %ax,%ax
   0x7c04:      mov    %ax,%ds
   0x7c06:      mov    %ax,%es
   0x7c08:      mov    %ax,%ss
   0x7c0a:      in     $0x64,%al
---Type <return> to continue, or q <return> to quit---
```

和 bootasm.S 的指令完全一致。

> 自己找一个 bootloader 或 kernel 中的代码位置设置断点并进行测试。

跟上述过程完全一致，只需更改断点的位置即可。

顺便，改写 Makefile 在启动 qemu 的时候加入 `-d in_asm -D q.log` 参数就可以将运行的汇编指令保存在 q.log 中。

## Assignment 03

> 分析 Bootloader 进入保护模式的过程。需要阅读 bootasm.S 的源码。

> 为何/如何开启 A20 ？

早期的 8086 处理器只有 20 根地址线，所以有些程序员在编写实模式的程序的时候使用了 F800:8000
表示 0x00000 的 trick 。
但是在 80286 中寻址空间增加到了 16MB ，不再是 1MB ，导致的结果就是第 21 根地址线 A20
在使用地址 F800:8000 时被异常置 1 ，这就成了一个 bug 。
为了 fix 这个问题，IBM 的工程师在 A20 地址线的主板上增加了一个逻辑门，BIOS 在内存检查之后就把它关闭，
导致实模式程序在访问 1MB 之上的部分地址都会被 wrap 回来，强制使用 1MB 的地址空间。
在切换到保护模式之前，再开启 A20 门，这样就可以访问所有内存地址了。  
开启 A20 是通过 8042 控制器实现的，首先写入 port 0x64 告诉 8042 控制器要写入了，
然后写入 0xDF 到 port 0x60 。

> 如何初始化 GDT 表？

从引导区载入 GDT 表：

```
    lgdt gdtdesc
```

> 如何使能和进入保护模式？

置 CR0 寄存器的 0 bit 为 1 ，即进入保护模式。

代码如下：

```
# Switch from real to protected mode, using a bootstrap GDT
# and segment translation that makes virtual addresses
# identical to physical addresses, so that the
# effective memory map does not change during the switch.
lgdt gdtdesc
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0
```

## Assignment 04

> 分析 bootloader 加载 ELF 格式 OS 的过程。
> 阅读 bootmain.c ，调试 bootloader 和 OS 。

> bootloader 是如何读取硬盘扇区的？

readsect() 函数从 secno 扇区读取到 dst 位置。

相关代码如下：

```c
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```


> bootloader 是如何加载 ELF 格式的 OS 的？

相关代码如下：

```c
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }

	    struct proghdr *ph, *eph;

	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;

	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

## Assignment 05

> 实现函数调用堆栈跟踪函数，并回答问题。

通过不断修改 ebp eip 为 saved_ebp saved_eip 向上回溯，直到 ebp = 0 。为了避免深度过深，
增加了 STACKFRAME_DEPTH 参数。

输出信息的话，ebp[0] 就是 saved ebp ，ebp[1] 就是 saved eip ，ebp[2] ~ ebp[5] 就是可能的四个参数。
使用 print_debug 输出函数名信息。

> 最后一行各个数值的含义？

最后一行是 bootmain 函数使用的堆栈。

## Assignment 06

> 完成中断初始化的处理，并回答下列问题。

> 中断向量表一个表项多少字节？哪几位代表中断处理程序入口？

struct gatedesc 一共 64 bit ，8 字节。

```c
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```

前后 16 位构成 offset ，中间 32 位是段选择子，共同构成中断入口地址。

> 完成 idt_init 。

主要是使用宏函数 SETGATE ，见代码。

> 完成时钟中断计数输出 100 ticks 。

见代码，很简单。

## Challenge 01

首先在 init.c 里的 kernel init 过程中加一个 INT 操作搞个中断出来：

```c
static void
lab1_switch_to_user(void) {
    //LAB1 CHALLENGE 1 : TODO
    asm volatile (
        "int %0 \n"
        "movl %%ebp, %%esp \n"
        :
        : "i"(T_SWITCH_TOU));
}
```

这是内核到用户态的代码。用户态到内核态的代码完全类似，只修改了中断描述符。

然后修改 trap.c 里的中断处理代码。

```
case T_SWITCH_TOU:
    if(tf -> tf_cs != USER_CS) {
        frame_k2u = * tf;

        frame_k2u.tf_cs = USER_CS;
        frame_k2u.tf_ds = USER_DS;
        frame_k2u.tf_es = USER_DS;
        frame_k2u.tf_ss = USER_DS;
        frame_k2u.tf_esp = (uint32_t)tf + sizeof(struct trapframe) - 8;
        frame_k2u.tf_eflags |= FL_IOPL_MASK;

        * ((uint32_t *)tf - 1) = (uint32_t) & frame_k2u;
    }
    break;
case T_SWITCH_TOK:
    // panic("T_SWITCH_** ??\n");
    if(tf -> tf_cs != KERNEL_CS) {
        frame_u2k = * tf;

        frame_u2k.tf_cs = KERNEL_CS;
        frame_u2k.tf_ds = KERNEL_DS;
        frame_u2k.tf_es = KERNEL_DS;
        frame_u2k.tf_ss = KERNEL_DS;
        frame_u2k.tf_esp = (uint32_t)tf - sizeof(struct trapframe) + 8;
        frame_u2k.tf_eflags &= ~FL_IOPL_MASK;

        * ((uint32_t *)tf - 1) = (uint32_t) & frame_u2k;
    }
    break;
```

这样就可以了。`make grade` 可通过全部评测。
