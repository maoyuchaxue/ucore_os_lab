# LAB3 REPORT

## 总述

本次实验是对实验二的补充，在实验二的基础上继续实现完整的页面管理功能。主要包括实现页面从磁盘中的换入换出（包含页面置换算法），以及缺页中断的处理。

## 练习

### 练习一/页错误中断处理

#### 实现细节

如lab2 report中所述，在出现缺页或出现异常时，会将相关虚拟地址存入cr2寄存器，错误信息位存入error code. 在本次lab中，我们对这一异常进行处理。

不过，实际上操作系统能处理的东西不多。对于访问、写入无权限的页这种异常，属于用户程序本身的错误，操作系统不可能让用户访问其权限外的数据。因此，我们在这里能处理的实际上只有页被换出或未被分配的错误。

我们首先使用lab2里实现的get\_pte获得页表项，之后分两种情况：

+ 当页表项为0时，表示此处无对应的物理页，即未分配。此时使用直接pgdir\_alloc\_page为之新分配一块内存即可。
+ 当页表项不为0，但页却不存在于内存中时，说明页已经被换出到外存当中，需要换入。用swap\_in换入页面，用page\_insert将内存页面与虚拟页面绑定，swap\_map\_swappable将该页面设计为允许换出。这样，页面换入的流程就结束了。

#### 题目回答

+ 请描述页目录项（Page Directory Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

页目录项和页表项的组成部分已于lab2中详述，这里只说对页替换算法的用处。

页表中有引用位，会在页面被引用时置为1；另外还有修改位，在修改时置为1. 这两者可以用于判断页的使用频率。定期收集各页的引用位，就可以得到页面在一段时间内的使用信息，也即是近似LRU类的置换算法的基本思路。在最不常使用的页面当中，又以未被修改的页面为最优先，这是一般对标志位的使用方法。

至于表项中本身包含的AVAIL位，只有3位，可扩展性不高而且太过依赖硬件，虽然可以使用但似乎并不好。

+ 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

按照x86手册，此时缺页异常中嵌套缺页异常，导致发生double fault；在double fault中基本不能进行任何措施，只会调用异常处理程序，处理程序结束后的运行情况是未定义的，无法继续运行下去。这种情况仅会发生在操作系统存在bug时。

#### 对答案

和答案相比，我的代码在处理异常情况时没有进行周全的考虑。答案在调用每个可能会失败的函数时（如因内存耗尽而无法分配新内存页面），都验证了调用是否成功，即返回值是否为NULL，并根据返回值显示错误信息并告知错误。而我没有对这些特殊情况进行判断，可以说答案更加严谨。

#### 知识点

这里涉及到的知识点有：

+ 页缺失异常的细节；
+ 页缺失的处理方法；
+ 页的换入换出。

### 练习二/FIFO页面替换算法

#### 实现细节

FIFO替换非常简单，将存在在内存中的所有页组织成一个队列——当然是用双向链表模拟的队列，每次添加页面时入队、需要换出页面时出队即可。

为了维护这个链表，按照本实验框架的链表设计，需要在页面Page结构中添加相关的list\_entry字段，用来存储链表信息。

也即，入队时将页插入到链表中的尾端，出队时选择链表的首端。这样无论哪一种操作都是O(1)的常数时间。

#### 题目回答

如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题：
+ 需要被换出的页的特征是什么？
+ 在ucore中如何判断具有这样特征的页？
+ 何时进行换入和换出操作？

回答：

并不需要修改框架，因为这一算法中所用的访问位和修改位都不需要软件进行设置，置换算法需要的功能完全可以在已有的api中实现，甚至不需要使用tick\_event这个api，因为不需要定时记录页面的使用情况。

为了实现extended clock算法，只需要在FIFO的基础上，设置一个标志当前指针位置的全局变量即可。当需要换出时，进行如下的操作：

+ 第一个检查的页不是直接取链表第一个元素，而是把当前指针沿着列表移动，开始进行遍历；
+ 首先遍历列表，试着找到访问位和修改位都是0的页；并在寻找过程中将经历的页进行修改：
    + 若遇到访问位修改位均为0的页，则该页符合要求，是所需的页；
    + 若遇到仅访问位为1的页，将访问位置为0；
    + 若遇到仅修改位为1的页，将修改位置为0并将它写回到磁盘上；
    + 若遇到访问位修改位均为1的页，则把访问位设为0。
+ 重复以上过程，直到找到符合要求的页为止。此时全局指针停在何处，下次即从这里开始算法。

算法即如上所述。

+ 需要被换出的页的特征是什么？

看情况决定，优先换出最近既未被访问、也未被修改的页；其次换出最近未被访问但被修改了，或是最近访问过但未修改的页，最后才换出最近访问过且修改的页。

+ 如何判断具有这样特征的页？

根据页表项的相关控制位（访问位，修改位）来判断。这些位由硬件设置，ucore只需要读取访问，换出程序在适当的时候把访问位、修改位重设为0即可。

+ 何时进行换入和换出操作？

当需要为一页分配物理帧，而该页已经之前被换出存在于磁盘上的时候，需要进行换入。当空间不足够，不能分配帧时，需要换出。

#### 对答案

本实验要实现的代码只有几行。和答案的不同还是有的：答案和我的代码在队列的方向上正好反了过来，我每次将新页插在环形链表的最后，而从链表的最前取出被换出的页；答案则将新页插在环形链表的最前，从链表的最后取出被换出的页，两者只是方向不同，但是环形链表是对称的所以没有问题。

#### 知识点分析

本练习牵涉到的知识点包括：

+ 具体的页面替换算法；
+ 页表项中和页面替换有关的控制位；

这里涉及的算法都是不需要定时收集使用情况的简单算法，感觉上过于简单了。不过，这类算法开销也比较小，对于ucore这样小型os来说是比较合理的。