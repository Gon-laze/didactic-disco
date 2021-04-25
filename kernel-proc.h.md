---
description: 进程相关定义
---

# kernel/proc.h

`kernel/proc.h`中存放了绝大多数进程相关的宏与数据结构，是整个操作系统中至关重要的一部分。

## 上下文/语境（context）

自然，一个操作系统不能也不应该从头到尾不间歇地执行下去。如果中途发生了严重的错误，系统必须能及时停止并返回原先状态。而如果中途需要处理更重要的事务，系统还要保存当前任务的状态以便处理完后继续工作。这样的异常及中断都涉及到寄存器的变化。

**context**的作用便是进行寄存器的清除及恢复。它保存了基本的数据寄存器（**s0~s11**），同时保存了返回时需要的地址**ra**及栈顶指针**sp**。xv6一般将其与`kernel/swtch.S`搭配来实现切换。

```c
// 为系统上下文切换保存寄存器
struct context {
  uint64 ra;
  uint64 sp;

  // 保存被调用方数据
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

## CPU/hart

**CPU**为处理进程的基本的单元，但和我们平常理解的CPU略有不同。现实中的CPU指一个处理器件，内部可能有多个处理核。而在这里，**CPU**指的是**一次处理一条指令的单线程单元**，而不是特指某个硬件结构。官方文档中有时也会出现**核（core）**，**硬件线程（hart/hard thread）**的术语，它们实际上指的是同一件事物。

`struct CPU`给出了当前**CPU**信息的一个抽象表示。

```c
// 各个cpu状态.
struct cpu {
  struct proc *proc;          // 正在该cpu上运行的进程，若该指针为空则空闲.
  struct context context;     // cpu当前的上下文/语境，进程调度由scheduler在此操作.
  int noff;                   // Depth of push_off() nesting.
  int intena;                 // Were interrupts enabled before push_off()?
};
```

这些信息被保存在`CPUS`中，方便查询。数目上限为`NCPU`（=8）。

```c
extern struct cpu cpus[NCPU];
```

## 陷阱帧（trapframe）

## 进程（process）

接下来的部分为本节最后也是最为关键的部分——**process**。

一个**process**需要提供以下内容：

* 便于识别与区分的身份信息：id号码...
* 掌握的内存资源：代码的运行空间，使用的文件，占用大小，当前内存指向...；
* 进程的状态以及相互关系：进程状态，父进程，是否被杀死...；
* 调度接口：上下文，等待链，返回状态，锁...

```c
/*
Per-process state（处理进程状态）
xv6 使用结构体 struct proc 来维护一个进程的众多状态。
*/
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state（指示了进程的状态：新建、准备运行、运行、等待 I/O 或退出状态中。）
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID（pid，进程号）

  // proc_tree_lock must be held when using this:
  struct proc *parent;         // Parent process

  // these are private to the process, so p->lock need not be held.

  // 每个进程有两个栈：用户栈和内核栈(kstack)。当进程在执行用户指令时，只有它的用户栈在使用，而它的内核栈是空的。
  // 当进程进入内核时（为了系统调用或中断），内核代码在进程的内核栈上执行；
  // 当进程在内核中时，它的用户栈仍然包含保存的数据，但不被主动使用。
  // 进程的线程在用户栈和内核栈中交替执行。内核栈是独立的（并且受到保护，不受用户代码的影响），所以即使一个进程用户栈被破坏了，内核也可以执行。

  uint64 kstack;               // Virtual address of kernel stack（内核栈的虚拟地址）
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```



