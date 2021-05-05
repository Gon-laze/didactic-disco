---
description: 简单定义了一个自旋锁
---

# kernel/spinlock.h

锁无疑是实现互斥的最好工具，尽管它不是最安全高效的，但它确实用了非常简单的手段来满足多程需求。xv6-riscv里共有两种锁：**自旋锁（spinlock）**和**睡眠锁（sleeplock）**，而自旋锁又是锁中较为简单的实现方案。

```c
// xv6 2种锁之一：自旋锁
// 没有抢夺到锁的进程将会一直尝试，简单地实现了互斥
struct spinlock {
  uint locked;       // 是否上锁？

  // 供调试部分
  char *name;        // 锁名
  struct cpu *cpu;   // 当前持有这个锁的cpu
};

```

整个结构非常简单，关键部分只有一个`locked`，当这个变量为真的时候表示已锁定。至于`name`和`cpu`则是方便调试的部分，并不需太过在意。

