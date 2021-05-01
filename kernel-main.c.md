---
description: 多核下初始化与运行
---

# kernel/main.c

当`kernel/start.c`完成了它的工作后，它会触发一个中断，使得机器由M模式下降到S模式，并正式开始系统的初始化与运行部分。

必须注意的一点是，xv6-riscv支持**多核系统**，这一点在本文件中尤为明显。实际上，`kernel/main.c`中的多核处理思路，无论是初始化还是运行方面，对系统性能有都着极其巨大的贡献。

## 系统初始化

如果我们的机器上不止有一个CPU，提高机器性能的最佳方案肯定是让它们全部处于工作状态；而为了让它们顺利开始工作，一些准备工作是必不可少的。每个CPU都需要先完成它之前的准备，然后才能正式进行处理阶段。

然而有一个问题：并非所有的准备工作需要每个CPU各自完成一遍，例如系统计时器所提供的时间信息可供所有的CPU使用；另一方面，这些准备工作也不允许完成多次，倘若每个CPU都自己独享一个计时器，各自拥有一套时间标准，那么一个需要在特定时刻终止的进程移交给了不同的CPU后，势必引发大规模的混乱。

一个理想的解决方案就是找出一个特定的CPU，让它首先完成共有的准备工作（顺便完成自己的准备工作）；在这之后，其他的CPU才被允许执行它们的初始化过程。xv6-riscv即采用这一种思路：CPU\(0\)处理先决条件，其他CPU随后各自完成它们的任务。

多亏了riscv指令架构，获取当前CPU信息并不是一件非常困难的事情，因为一个专用的寄存器保存了相关的信息。`cpuid()`对其做了一个简单的包装 ，最终能返回当前CPU编号。

初始化工作非常繁琐。对于整体，需要做的工作有：

* 清理内存；
* 为内核代码创建、初始化页表；
* \(waiting......\)

```c
if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode cache
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process（开始创建用户进程）
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }
```

## 系统运行

等到所有的准备工作都完成了以后，CPU将开始进入运行状态。运行的过程非常简单，只有一个函数`scheduler()`完成：

```c
    scheduler();        
```

`scheduler()`首先会对当前CPU的进程信息记录进行清理。主体部分运行在一个`for(;;)`循环中，保证开机过程一直运行。接着，它会持续尝试在进程槽位`proc`中寻找状态为**RUNNABLE**的条目，一旦找到便加载到CPU，并在结束后重新清理CPU进程信息，如此轮番调度完成进程的处理。

对scheduler\(\)的更详细信息，参阅`kernel/proc::scheduler()`。

