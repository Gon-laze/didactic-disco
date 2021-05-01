# kernel/proc.c

## 基本架构

我们在`kernel/proc.h`中介绍了**process**和**CPU**的表示，它们两个构成了进程运行部分的核心，本文件大部分内容都围绕此展开。

```c
struct cpu cpus[NCPU];    //最多允许NCPU个CPU

struct proc proc[NPROC];  //最多允许NPROC个进程存在（应该是整个机器，而不是单个CPU）

struct proc *initproc;

int nextpid = 1;
struct spinlock pid_lock;
```

`CPUS`和`proc`可以理解为**CPU**和**process**的总体集合/槽位，拥有数目上限，软件层面对**CPU**和**process**的查询都是在其中查找（底层还有其他手段）。

`initproc`为初始化程序，可以理解为进程树结构最上方的空结点。

进程的**pid**取决于`nextpid`。每次创建新的进程的**pid**都将被赋予当前`nextpid`的值，随后`nextpid`将自增。`pid_lock`则保证操作的原子性。

```c

extern char trampoline[]; // trampoline.S

// 帮助确保等待中的父进程的唤醒不会丢失。
// 避免访问父进程时的内存异常
// 必须在任何p->lock之前获得。
struct spinlock wait_lock;
```

`trampoline[]`实际上就是`kernel/trampoline.S`的地址，它负责实现中断跳转处理等工作。

{% hint style="info" %}
在链接过程中，编译器将会自动寻找与声明同名的定义，若恰好有一个同名，则进行匹配。这个过程并不会被声明的类型影响，无论是`char`还是`int`；另外，数组类型告诉编译器使用时视作一个地址，而不是一个值。最终`trampoline[]`中只会存放找到的那个唯一定义，即`kernel/trampoline.S`的地址。

然而有一点：为什么要这么干？考虑到我们要尽可能高效地实现内核，尤其是运转最多部分的代码，采用无冗余的汇编码肯定不会逊色于C语言；另外，如果我们要给硬件访问`kernel/trampoline.S`中某函数（如**uservec**）的接口，且已知`kernel/trampoline.S`在内存中存放的位置**TRAMPOLINE**，一个理想的解决方案便是计算出**uservec**相对于源文件的偏移量**offset**，再将 $$TRAMPOLINE+offset$$ 这个地址暴露给硬件，而**offset**在`char`类型下很容易由$$uservec-trampoline$$ 得到。

总之，这种声明至少证明调用函数并不一定要函数名或者函数指针，甚至`char[]`也能达到目的（更甚至效果还相当不错）。无论是实用性还是优美性来说，这都是一个非常有意思的案例。
{% endhint %}

`wait_lock`保持父进程的唤醒过程，避免异常。

## 进程内核栈处理

前面提到，每一个进程都有用户栈和内核栈。`proc_mapstacks()`即完成这一分配工作，它将在RAM中申请页面，并映射于虚拟地址的高处；随后，`procinit()`把申请好的页面分配给`proc`中的槽位。

### 栈映射（proc\_mapstacks\(\)）

```c
// 为每个进程的内核堆栈分配一个页面
// 将其映射到内存中的高位
// 之后跟随一个无效的防护页。
void
proc_mapstacks(pagetable_t kpgtbl) {
  struct proc *p;
  
  for(p = proc; p < &proc[NPROC]; p++) {
    char *pa = kalloc();
    if(pa == 0)
      panic("kalloc");
    uint64 va = KSTACK((int) (p - proc));
    // 由于 pa 是kalloc()于RAM（0x80000000后）中分配的页，而va是栈顶位置
    // 这一步即实现了物理内核位置映射到虚拟栈顶
    kvmmap(kpgtbl, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  }
}
```

### 栈初始化（procinit\(\)）

```c
// 将由proc_mapstacks()分配的kstack交给各个进程
void
procinit(void)
{
  struct proc *p;
  
  initlock(&pid_lock, "nextpid");
  initlock(&wait_lock, "wait_lock");
  for(p = proc; p < &proc[NPROC]; p++) {
      initlock(&p->lock, "proc");
      p->kstack = KSTACK((int) (p - proc));
  }
}
```

## CPU/进程查询

了解处理当前进程的CPU编号是非常必要的，因为编号是区分CPU的唯一标识。当处理进程互斥问题的时候，，必须要了解CPU编号。

riscv提供了特殊寄存器**tp（thread pointer）**来保存当前硬件进程的编号，通过`kernel/riscv.h::r_tp()`进行访问。`cpuid()`对此进行了一个简单包装，之后可交给`mycpu()`在`cpus`中查询对应信息，其中当前进程信息可由`myproc()`返回。

{% hint style="warning" %}
读取当前CPU信息必须保证这个过程不会交换给其他CPU处理，因此一定切记关闭中断！
{% endhint %}

### CPU编号（cpuid\(\)）

```c
// 返回当前cpu的编号
// 必须于无中断时调用，避免进程移交给不同的CPU
int
cpuid()
{
  int id = r_tp();
  return id;
}
```

### 当前CPU（mycpu\(\)）

```c
// 返回当前cpu的状态指针
// 必须于无中断时调用，避免进程移交给不同的CPU
struct cpu*
mycpu(void) {
  int id = cpuid();
  struct cpu *c = &cpus[id];
  return c;
}
```

### 当前进程（myproc\(\)）

```c
// 返回当前cpu上正在运行的进程信息
// 为0则cpu空闲
struct proc*
myproc(void) {
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```

## 分配与释放



