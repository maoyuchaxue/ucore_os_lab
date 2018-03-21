# LAB4 REPORT

## 总述

lab4主要实现的是进程的创建，以及最基本的进程切换功能。本lab还没有载入外部程序的功能，因此实际上创建的进程只有idleproc(在无其他进程运行时默认运行的进程)和initproc(打印Hello world表示进程切换成功)这两者。

## 实验

### 练习一/进程初始化

描述一个进程信息的结构proc\_struct的定义如下所示：

```
struct proc_struct {
    enum proc_state state;                      // Process state
    int pid;                                    // Process ID
    int runs;                                   // the running times of Proces
    uintptr_t kstack;                           // Process kernel stack
    volatile bool need_resched;                 // bool value: need to be rescheduled to release CPU?
    struct proc_struct *parent;                 // the parent process
    struct mm_struct *mm;                       // Process's memory management field
    struct context context;                     // Switch here to run process
    struct trapframe *tf;                       // Trap frame for current interrupt
    uintptr_t cr3;                              // CR3 register: the base addr of Page Directroy Table(PDT)
    uint32_t flags;                             // Process flag
    char name[PROC_NAME_LEN + 1];               // Process name
    list_entry_t list_link;                     // Process link list 
    list_entry_t hash_link;                     // Process hash list
};
```

这其中的一些字段应该在结构被分配内存时就被初始化：

+ 待设置的指针(mm, tf, parent)应该被设为NULL;
+ 状态应该被设为PROC\_UNINIT;
+ 控制位(runs, flags, need\_resched)应该被初始化：runs=0, flags=0, need\_resched=0;
+ cr3应被设为内核的cr3;

还有一些字段是应当在创建进程后被设置，否则无法正常运行的，比如pid, name, 这些字段其实我们是否在这里进行默认初始化的影响也不大，因为是一定要被立即由其他代码初始化的。

#### 问题回答

+ 请说明proc\_struct中`struct context context`和`struct trapframe *tf`成员变量含义和在本实验中的作用是啥？

首先观察`struct context`的定义:

```
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

可见，这里保存的是进程运行中的寄存器状态。当进程间互相切换时，为了能让进程自身感受不到发生过切换，在离开进程、进入进程时需要保存、恢复寄存器。这里的context即用于寄存器信息的存储。

此外，`struct trapframe *tf`原本是中断中用于储存原位置执行信息，在中断结束后负责还原现场、返回原地址的结构。而此处是借用来切换特权级、进入子进程的结构体。在进程进行fork时，新被创建进程中将会设置一个trapframe，存储在进程栈顶；同时，新进程的eip被设为forkret这个函数。这个函数实际上会进入中断处理函数的还原现场、返回原执行位置的部分。相当于原本没有中断，而我们既伪造了中断现场，又伪造了中断返回过程，这样就和中断结束后一样，可以正确开始新进程的执行。

使用`tf`的主要原因是子进程需要运行在不同的特权级，这一点用iret+trapframe处理最为简单合理。

#### 对答案

和答案相比，发现自己有一些字段没有初始化，如context和name。不过感觉上，context会在线程运行的过程中被初始化（先是fork时设置eip、esp,然后trapret时按trapframe设置其他寄存器、在退出或被调度时这些值存入context），而name也会在进程被建立时设置，所以应该不会导致问题。

### 练习二/新内核线程资源分配

#### 实现细节

在ucore中，线程实际上只是另一个进程而已，只是与产生出它的进程共享一个内存空间。

创建新内核线程时，需要进行以下步骤：

+ 调用alloc\_proc创建一个空进程；
+ 调用get\_pid获得不重复的进程号，并赋给新进程；
+ 调用setup\_kstack设立进程栈；
+ 调用copy\_mm复制或共享内存管理信息(mm)；（实际上这次lab并没有用到，因为都是内核进程）
+ 调用copy\_thread复制进程上下文(设置tf以及context字段)；
+ 调用hash\_proc, list\_add，将进程添加到哈希表、链表中。
+ 调用wakeup\_proc，设置进程的状态为PROC\_RUNNABLE.
+ 至此，新线程基本建立完毕，返回即可。

#### 问题回答

+ 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

是的，ucore可以保证当前依然存在的进程中pid互不重复。这一点由get\_pid函数保证:

```
static int
get_pid(void) {
    static_assert(MAX_PID > MAX_PROCESS);
    struct proc_struct *proc;
    list_entry_t *list = &proc_list, *le;
    static int next_safe = MAX_PID, last_pid = MAX_PID;
    if (++ last_pid >= MAX_PID) {
        last_pid = 1;
        goto inside;
    }
    if (last_pid >= next_safe) {
    inside:
        next_safe = MAX_PID;
    repeat:
        le = list;
        while ((le = list_next(le)) != list) {
            proc = le2proc(le, list_link);
            if (proc->pid == last_pid) {
                if (++ last_pid >= next_safe) {
                    if (last_pid >= MAX_PID) {
                        last_pid = 1;
                    }
                    next_safe = MAX_PID;
                    goto repeat;
                }
            }
            else if (proc->pid > last_pid && next_safe > proc->pid) {
                next_safe = proc->pid;
            }
        }
    }
    return last_pid;
}
```

这一段代码中，有两个静态变量`last_pid, next_safe`，分别用于维护上一个分配的pid，以及其他进程中大于且最接近last\_pid的pid，也即是在\[last\_pid, next\_safe\)之间的pid都是可以使用的。

这段代码中有两种情况: 如果`last_pid+1 < next_safe`, 那么就分配`last_pid+1`即可。而若突破了next\_safe，则需要更新这两个值；

更新这两个值的过程由于用了goto看上去比较复杂，不过基本上可以组织为两个步骤：第一轮遍历全部的进程，不断地把last\_pid自增，直到不和进程里其他pid重复为止；此时把next\_safe设为MAX\_PID, 然后再进行第二轮遍历：对每个进程`proc`, 如果`last_pid < proc->pid < next_safe`, 则让`next_safe = proc->pid`，如此即让next\_safe成为了所有进程中大于当前pid且最接近的一个，满足了定义的性质。

看上去需要两轮功能不同的遍历，不过程序里强行把两轮遍历合在了一个while循环里，如果在遍历中遇到了重复的元素，这次循环就归于第一轮；若没有遇到重复的元素，则归于第二轮。

综上，除非进程多到已经没有pid可以分配，否则是可以保证pid互不重复的。

#### 对答案

和答案进行对照，发现自己的代码有一些疏漏之处。答案在获得pid，将进程插入进程列表中时，进行了关中断操作，然后在插入完毕时又开中断。这样可以保证进程的插入不会被中断打断，更加合理，否则可能会导致两个进程创建时因为被调度而同时调用这一段代码，产生数据结构上的问题。

另外，答案对每个步骤中调用api可能产生的错误都进行了判断处理，例如无法给进程分配空间、无法给进程分配栈、无法给进程分配内存管理结构(mm)等，都会将已经分配出的空间释放掉然后返回。这一点上答案想的更加周全。

#### 知识点分析

这里牵涉到了内核线程创建的相关知识，包括pid的分配，内存等结构的复制等。重点在于如何指定线程的入口、初始状态。

课件上说了多种线程的实现方式，但显然ucore之中只能选择其中一种实现，所以相应的也只能覆盖到这一种实现的知识点，还是很不全面的。

### 练习三/分析进程起始与切换

#### proc\_run的分析

proc\_run的核心代码实际上如下：

```
struct proc_struct *prev = current, *next = proc;
local_intr_save(intr_flag);
{
    current = proc;
    load_esp0(next->kstack + KSTACKSIZE);
    lcr3(next->cr3);
    switch_to(&(prev->context), &(next->context));
}
local_intr_restore(intr_flag);
```

其中，`local_intr_save, local_intr_restore`的功能在之后的问题回答中详述。这里主要说明其他部分的功能。

首先load\_esp0将TSS中的特权态0下栈顶指针设为了`next->kstack + KSTACKSIZE`，这正是next进程的栈顶。这样，当进程中发生特权级切换时，就会将中断产生的trapframe结构压在此处，正好让esp到达`proc->tf`所指的位置，于是便可以不修改这个指针、每次都用来处理进程中断。

lcr3是将cr3设为next进程的cr3，也即切换页表，使得不同进程的不同页表可以正常工作。

switch\_to首先是把当前的寄存器值保存到了当前进程的context结构中，然后又把下一个进程的context里保存的寄存器值读出来(包括eip)，并且跳转到下一个进程开始执行。这样，进程的切换就完成了。

#### 问题回答

+ 在本实验的执行过程中，创建且运行了几个内核线程？

创建了2个线程，一个是idle\_proc，是在无其他任务时默认运行的空闲线程，并不做实质上有意义的操作，只是不断试图寻找别的就绪进程来运行；另一个是init\_proc，仅负责打印Hello world表示实验成功。

+ 语句`local_intr_save(intr_flag);....local_intr_restore(intr_flag);`在这里有何作用?请说明理由

这两个指令分别负责关中断、开中断。由于这里涉及到了进程切换的操作，在完全切换完之前是不能发生中断的，否则就会导致巨大的问题：比如还没有跳转到另一个进程的执行地址，先切换了页表，在此时发生中断，导致无法正常处理，等等。

不过这里的细节在于，执行到中间`switch_to`函数时，实际上就已经跳转到下一个进程了；问题就在于需要确保下一个进程中打开中断，否则中断就被永远关闭了。解决这个问题，有两种情况;

+ 如果跳转到的下一个进程是刚刚进入，即刚开始执行时，此时其起始地址是forkret函数，追踪代码可知，它调用了中断的返回部分trapret.在这里使用了iret指令。因此，中断设置位被还原为进程中trapframe里所设的eflags里的相应标志位。而这里，`copy_thread`函数已经为新进程设置好了：`proc->tf->tf_eflags |= FL_IF;` 这样一来就没有问题了。
+ 如果跳转到的下一个进程是已经执行过，因调度而被切换的进程，也没有任何问题，因为`proc_run`函数是进程间切换的唯一入口，也即一个进程调度结束时，`switch_to`所记录的进程运行状态中，eip一定就在此处。因此，上一个进程执行完了`local_intr_save`, 下一个进程也随着执行`local_intr_restore`，仍然能够正常执行下去。

#### 知识点分析

这里涉及到的知识点是进程的切换，主要是通过阅读代码详细了解了进程间互相跳转、调度的过程。另外，对其中的细节(各种寄存器的赋值，环境的保存与恢复)进行了深刻详细的了解。

不过这里还只是进程管理的一半左右，真正实现进程的完整生命流程还得等到lab5完成。


