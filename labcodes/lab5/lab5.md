# LAB5 REPORT

## 综述

lab4里进行了进程的初始化、切换操作，但还很不完善：没有用户进程，也不能退出。在这次lab中我们继续完善已有的工作，实现用户程序的加载、进程运行整个过程。

## 实验

### 练习零/新功能的支持

为了和lab5的框架相兼容，lab4的代码需要进行两处修改。一处是初始化时，由于这一次增加了wait\_state, optr, cptr, yptr几个成员变量，也需要进行初始化；还有，lab4里只设了进程的父进程，但没有设置optr,cptr,yptr这几个用于访问子进程的变量，故在fork时需要使用set\_links这一api来完成设置。

### 练习一/应用程序加载

#### 实现之前

##### Makefile

首先我们需要了解ucore究竟是如何构建用户程序的。在Makefile中，进行了用户程序的编译，并且将其以二进制形式链接到了kernel中：

```
$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS) -b binary $(USER_BINS)
```

在ld中，经过链接，每个用户程序会生成`_binary_obj___user_proc_out_start, _binary_obj___user_proc_out_size`两个符号，其中`proc`为用户程序名。通过这两个指针，就可以得到用户程序的起始地址和长度。

##### 调用用户程序

在上一个lab中什么都没有做的init\_proc在这个lab里调用了用户程序。具体执行部分如下：

```
// user_main - kernel thread used to exec a user program
static int
user_main(void *arg) {
#ifdef TEST
    KERNEL_EXECVE2(TEST, TESTSTART, TESTSIZE);
#else
    KERNEL_EXECVE(exit);
#endif
    panic("user_main execve failed.\n");
}
```

这里，如果使用`make run-xxx`指令来指明执行某一用户程序，则在makefile中会加入对`TEST, TESTSTART, TESTSIZE`的定义：

```
build-%: touch
    $(V)$(MAKE) $(MAKEOPTS) "DEFS+=-DTEST=$* -DTESTSTART=$(RUN_PREFIX)$*_out_start -DTESTSIZE=$(RUN_PREFIX)$*_out_size"
```

而若不特意标注，则执行exit这个默认的用户程序。

随后，ucore使用宏把这个调用落实到C代码，如下：

```
static int
kernel_execve(const char *name, unsigned char *binary, size_t size) {
    int ret, len = strlen(name);
    asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL), "0" (SYS_exec), "d" (name), "c" (len), "b" (binary), "D" (size)
        : "memory");
    return ret;
}

#define __KERNEL_EXECVE(name, binary, size) ({                          \
            cprintf("kernel_execve: pid = %d, name = \"%s\".\n",        \
                    current->pid, name);                                \
            kernel_execve(name, binary, (size_t)(size));                \
        })

#define KERNEL_EXECVE(x) ({                                             \
            extern unsigned char _binary_obj___user_##x##_out_start[],  \
                _binary_obj___user_##x##_out_size[];                    \
            __KERNEL_EXECVE(#x, _binary_obj___user_##x##_out_start,     \
                            _binary_obj___user_##x##_out_size);         \
        })
```

最后，由宏指定的调用变成一个系统调用，指定好了的参数被作为系统调用的参数传入。在这之后，就完全交给C代码来处理了。

#### 实现细节

系统调用响应时可以说直接调用了load\_icode这个函数，它负责把用户程序载入，并构造出一个用户进程。

load\_icode函数的步骤比较漫长————但是在这里清清楚楚地分析掉是有好处的，因为在lab8中我们还要从文件系统自行实现一遍这个过程。虽然在获得数据的方式上有略微的不同，不过主要的处理方法没有变。

步骤如下:

+ 首先创建一个新的`mm_struct`结构体，用于安排用户进程的页面；
+ 为内存管理进行初始化，建立页目录表；
+ 把elfhdr结构指针指向用户进程数据起点，就可以开始解析elf文件结构了；
+ 首先比较`elfhdr->e_magic = ELF_MAGIC`，确认elf文件格式正确；
+ 对`elfhdr->e_phoff`开始的一系列`proghdr`结构分别进行解析:
    + 根据`proghdr->p_va, proghdr->p_memsz`确定连续的虚拟内存空间，把这些空间加入到vma列表中；
    + 将proghdr下的数据以页为单位，分配空间，并拷贝至相应的分配出的页中，完成连续物理存储到非连续页式存储的转换。
+ 为马上要运行的进程分配栈：将栈的地址空间在vma结构中注册，并且给栈分配最初的几个页面。
+ 为当前进程分配trapframe，准备用于进入时的跳转：（本练习主要要做的）
    + cs段寄存器设为USER\_CS; ds, es, ss设为USER\_DS;
    + esp设为用户栈顶(USTACKTOP);
    + eip设为`elfhdr->e_entry`，即程序入口；
    + eflags加上`FL_IF`，如lab4中所述的那样，需要保证能够开启中断。

#### 问题回答

接下来分析load\_icode之后程序的运行情况。

在load\_icode操作之后，实际上已经完成的有:

+ 把当前进程变成了用户进程，设置好了页表，以及对应的数据都已加载进页面中；
+ current进程的trapframe已经设好，经由其跳转即可真正改变权限、跳入入口地址，从而进入用户进程。

接下来，do\_execve的系统调用结束，程序继续进行，直至系统调用处理的最后从中断返回——此时之前设好的trapframe起到作用，各段寄存器被正确设置，进入用户态；esp转到用户态栈；eip指向程序入口地址，开始用户程序执行。

#### 对答案

和答案完全相同，毕竟注释写的太详细了，基本上就是在逐句描述代码，就差直接把代码写在上面了…………

#### 知识点

这里涉及到的知识点有:

+ elf文件格式，比lab1涉及的要更详细更全面；
+ 用户进程的建立，特权级的切换；
+ 用户程序的载入。


### 练习二/父子进程

在lab4中，由于全都是内核线程，所以`copy_mm`并没有起到任何作用。现在加入了内核线程，则需要确实地复制内存管理信息。观察调用链：

```
copy_mm -> dup_mmap -> copy_range
```

知最终完成内容复制的还是copy\_range函数，也即是我们需要实现的函数。

这个函数所做的很简单，即根据给出的地址，分别从源进程和目标进程的页表中查找出对应页，在内核态下将源页拷贝至目标页中。

#### 实现细节

只需要进行以下操作：

+ 根据要拷贝的虚拟地址，使用get\_pte函数找到对应的源页表项、目标页表项；
+ 使用pte2page函数，找到源页表项对应的页；使用alloc\_page函数，为目标页表项分配一个新页；
+ 使用page2kva函数，获得这两个页对应的内核态下虚拟地址；
+ 使用memcpy，将源页内容拷贝至内核态的目标页下；
+ 使用page\_insert,将目标页插入到目标进程的页表项中并设置地址对应关系。

这样，内存的拷贝就完成了。

#### 问题回答

+ 请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

Copy on Write是为了节约存储空间的一种内存共享机制，也即当需要多进程共享同一内存时，无需每片内存都重新拷贝一遍，而是暂时使用同一个内存区域；等到需要修改（即写入）时，再做区分——真正拷贝内存，把两个进程原本共用的内存分开至不同的内存区域中。

设计如下: 父进程创建子进程时，不全部拷贝内存，而是把子进程的页表项直接指向自己的物理页，但权限设为只读；当其要写入时，会发生page fault, 此时根据错误类型、进程间父子关系来判断是否属于Copy on Write情况，若是，则找出源物理页，真正按之前步骤进行复制。

再详细一些的设计：需要更改copy\_range和do\_pgfault两个函数。具体修改内容如下：

+ copy\_range里不要为目标页分配新页，而是直接设置目标页表项，其指向的地址设为源物理页，权限设为只读(无PTE\_W)；
+ do\_pgfault里添加对写入只读页的处理部分，在此处进行以下操作：
    + 获得其parent进程，查看该进程在发生page fault处的页表项；
    + 若其父进程和该进程在page fault处页表项指向同一片地址，表示需要进行copy on write;
    + 为子进程新分配一个权限完全的页面，按旧版copy\_range相同的方式拷贝内存、设置到子进程页表项中。
    + 这样一来，子进程获得了复制后的内存，且拥有权限，可继续执行下去。

#### 对答案

同样，注释已经把代码一行行描述出来了，和答案只有变量命名上的差别。

#### 知识点

涉及到了以下知识点：

+ 子进程的创建；
+ 进程的fork过程；
+ Copy on Write机制。

### 练习三/进程相关系统调用

#### 程序分析

##### 系统调用的实现

系统调用实际上和中断共用了处理程序。和中断有区别的地方在于，系统调用需要传入参数、需要返回值。其传入参数的方式是向eax, ebx等寄存器依次存入参数，返回值也保存在系统调用后的eax中。

与普通中断不同的是在trap\_dispatch之后，处理程序根据系统调用号，选择具体的系统调用处理函数(如sys\_exit, sys\_wait等)，将trapframe中保存的寄存器信息读出作为参数。除此之外，都只是C语言实现的细节了。

##### fork

fork的机制在lab4里已经分析的比较透彻了，包括创建新进程、分配pid、设置用户栈、复制进程内存信息等等。fork结束之后，并不是马上进入新进程开始运行，而只是把新进程唤醒为RUNNABLE状态，等待调度。

##### exec

exec即练习一中分析的一整个流程，它从物理内存空间中读出用户程序并加载到当前的进程下，并且切换到用户态，把当前的进程完全替换成用户进程。这个系统调用不会再返回源程序了，而是直接进入替换了的新进程开始执行。

##### wait

wait可以等待进程号为pid的子进程结束。若pid指定为0，则会等待任意一个子进程结束。

在do\_wait中，首先检查等待的子程序是否已经完成（即处于ZOMBIE态），如果未完成，则把自己设为WAITING态，并且让出cpu，调度其他进程执行。调度完毕，这个父进程再次被激活时，则跳转回开头再次检查子程序；等到子程序已完成后，首先将该进程的返回码存入自己的返回值中，然后从进程表（hash表+链表）中去除这个子进程，同时清空其栈空间、最后清空这个进程结构，将其彻底从内存中清除。

##### exit

进程退出时，进行如下的操作：

+ 首先将cr3设为内核下的状态，因为马上要回收旧进程的内存管理信息，并且可能会删除其中的页表。
+ 减小内存管理信息mm的引用，如果发现引用数已经为0，即没有其他进程与其共享，则可以回收内存管理信息，包括页表、vma、页目录、最后是管理信息自身。
+ 设置当前进程状态为ZOMBIE，设置返回码为exit调用传入的参数。
+ 查看其父子进程。对其父进程，如果其正在等待当前进程，则唤醒之，告知其等待已经可以结束；对于子进程，不能直接结束，而是将其所有子进程移到initproc下，成为initproc的子进程。
+ 调度其他进程运行。由于本进程已经被设为了ZOMBIE态，不会再被调度执行，因此到这里就再也不会回到此进程了，进程生命结束。

#### 问题回答

+ 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

其过程已如上所述。fork新建一个进程，exec替换当前进程并直接进入用户程序执行；wait将自己不断设为等待态，并清空处于ZOMBIE态的子程序；exit将自己设为ZOMBIE态并结束进程运行。

+ 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

根据以上的分析，状态切换图大致如下：


>          ​                  (proc_init)          (do_wait)
>          PROC_UNINIT ---> PROC_RUNNABLE ---> PROC_SLEEPING 
>          ​                 |          ↑  (wakeup_proc)   |
>          ​       (do_exit) |          +------------------+
>          ​                 ↓
>          ​            PROC_ZOMBIE

#### 知识点总结

本部分涉及的知识点比较多：

+ 系统调用的原理；
+ 进程基本的三状态模型；
+ 进程状态间切换过程。

到这里为止一个简单的，具有基本功能的进程模型就被建立起来了。接下来的几个lab会进一步完善这个结构，来完成带优先级的进程调度、带阻塞的完整进程模型。
