# LAB6 REPORT

## 综述

本lab实现了进程带优先级的调度。总共涉及了Round Robin和Stride Scheduling两种调度算法。

这个Lab对架构的改动比较小，站在lab5的角度上实际上只改了schedule, wakeup\_proc和时钟中断的实现。因为只有这些涉及到了进程队列的调度，本次实验再次使用函数指针实现了一个类似面向对象的调度算法结构，使得算法只需实现对外的api，而无需担心它被调用的问题。

## 实验

### 练习零/兼容性修改

为了让lab1-5的代码与lab6相容，需要进行一些修改:

+ 在进程创建时，需要同时对其优先级、lab6\_stride等等进行默认初始化。
+ 在时钟中断处理时，应当调用`sched_class_proc_tick`函数，使用lab6的框架实现调度。另外，之前的lab定时将ticks清零，从这个lab开始用户程序需要获取ticks的实际数值，所以不能再清零了。

### 练习一/Round Robin观察

#### 代码分析

##### API调用分析

sched\_class结构体包含以下的函数：

```
struct sched_class {
    // the name of sched_class
    const char *name;
    // Init the run queue
    void (*init)(struct run_queue *rq);
    // put the proc into runqueue, and this function must be called with rq_lock
    void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
    // get the proc out runqueue, and this function must be called with rq_lock
    void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
    // choose the next runnable task
    struct proc_struct *(*pick_next)(struct run_queue *rq);
    // dealer of the time-tick
    void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
};
```

即包括init, enqueue, dequeue, pick\_next, proc\_tick五个函数。我们分别分析它们在何处被使用：

+ init在sched\_init中调用，用于初始化调度算法；
+ enqueue在sched\_class\_enqueue中调用，有两个入口：
    + 在wakeup\_proc中，会调用此函数将被唤醒的进程加入待调度队列；
    + 在schedule中，会调用此函数将当前执行的进程（由于接下来要调度，所以当前进程会停止执行）加入待调度队列。
+ pick\_next在sched\_class\_pick\_next中调用，只有一个入口：在schedule中，用于选择下一个调度的进程。
+ dequeue在sched\_class\_dequeue中调用，只有一个入口：在schedule中，选择完下一个调度的进程时，将它从进程队列中删去。
+ proc\_tick在sched\_class\_proc\_tick中调用，唯一作用是在时钟中断发生时，根据当前进程已执行了的时间片判断是否应当停止运行、将cpu交给下个进程。

##### 调度过程分析

当一个进程开始执行后，在不出错误正常执行时，只有一种方式来结束这个进程的执行、转移到其他进程，就是调用schedule方法。这会发生在进程退出、进程等待、根据时钟统计已经到达算法所允许的执行时间上限这几种情况下。

在调度时，会根据pick\_next的返回结果，从队列中删去这一进程（相当于进入执行态），而当该进程运行暂停需要让位给其他进程时，则将其再次入队。这样，round robin的实现就更简单了：不需要维护进程的循环队列中当前执行的位置，因为被执行了的进程会自动退出队列、重新进入。这样，每次进程入队时加在队尾，出队时选择队首进程的出队，就自然完成了RR算法。

#### 问题回答

+ 请理解并分析sched_class中各个函数指针的用法，并接合Round Robin调度算法描述ucore的调度执行过程.

这在上面已经全部说明。

+ 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

多级反馈队列算法(MFQ)即有多个优先级不同的调度队列；高优先级的队列限制最大时间片，由高到低时间片限制逐渐增加，执行到达时间片限制时，进程被存入较低一级队列运行。直到被转入最底层队列，这层队列按照FCFS运行，即不再根据时间调度进程。

为了实现，首先需要修改run\_queue结构，在其中存储多个调度队列的链表头部；并修改proc\_struct结构，添加一个int项(如`proc_struct->mbr_priority`)注明其当前处于哪个调度队列中，初始化为第一个队列。

之后，需要新建一个sched\_class结构，即负责MFQ算法的调度程序结构。其各函数的具体实现如下：

+ init中初始化多个调度队列；
+ enqueue中根据`proc_struct->mbr_priority`指定的优先级，将进程插入到指定队列的队尾；
+ dequeue时，根据`proc_struct->mbr_priority`，若非已处于最低优先级，则`proc_struct->mbr_priority--`；
+ pick\_next处，根据优先级从高到低，选取优先级最高的非空调度队列的队首进程作为下一个执行的进程；
+ proc\_tick中，根据当前所处队列判断是否时间片到达限制，若是，置`need_resched=1`表示应当调度之。

至此，MFQ算法就完全实现了。

### 练习二/Stride Scheduling算法

#### 算法实现

结构框架和RR算法没有区别，只是内部实现修改了。因此这里只说明Stride Scheduling(SS)算法本身的实现了。

这里我们使用了斜堆数据结构来实现优先级队列，这一结构的优点是操作简单、速度快，插入、删除都被归于合并操作中。

SS算法的核心是根据优先级为每个进程设置stride表示进程进度，每次执行时选择stride最小的进程执行，执行之后给进程的stride进行更新:`stride += BIG_STRIDE / proc->lab6_priority`.

因此，在斜堆数据结构已经实现之后，要做的就比较简单了：

+ 在init中初始化斜堆结构为NULL;
+ enqueue中将新进程插入斜堆中，置进程的时间片为max\_time\_slice;
+ dequeue中从斜堆中移除当前进程节点(skew\_heap\_remove);
+ pick\_next处选取斜堆的根节点(即stride最小节点)返回，并更新其stride;
+ proc\_tick和RR算法、MFQ算法处理相同，仅等待时间片用完、然后设置重新调度。

#### 对答案

和答案对比，发现答案根据`USE_SKEW_HEAP`宏，写了使用斜堆、使用普通队列的两种不同实现，而我只实现了斜堆的这一种。相比之下，普通队列的插入删除均为常数时间，但调度进程复杂度为O(n); 斜堆任何操作均为O(log n)时间，应当认为斜堆效率更高。此外，实际上究竟使用什么数据结构也无法在操作系统运行时修改。因此感觉写普通队列实现的意义并不大。

另外答案考虑到了`priority=0`时等价于`priority=1`，避免发生除以0的错误，这样更加周全一些。但是stride scheduling的定义中priority相当于运行时间比例，设置为0没有意义。应当怎么处理priority=0的情况应该是未定义的。

### 知识点总结

本次实验的总体内容比较少，涉及了进程的调度算法。由于框架体量问题，实验内只涉及了3种算法，而实际上存在的实时调度、多处理器调度算法等等都并不能涉及。因此，还是需要课本/视频上概念的学习。