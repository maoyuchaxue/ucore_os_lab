# LAB8 REPORT

## 综述

本次lab为ucore添加了一整个文件系统——包括vfs, sfs, 设备层完整的三层结构。本次实验新增的功能在架构上对操作系统的改动比较大，新增的文件模块的规模也很大，和需要实际自己写的代码相比，总体需要阅读的代码量要更多。因此，我们先分析文件系统的整体架构。

## 代码阅读

### 工程构建

本次lab中，需要一个与SFS文件系统相匹配的硬盘来存储用户程序。为此，在makefile中改写了对用户程序的编译，将用户程序分别编译之后，使用了mksfs.c作为工具，来从一个工程下的目录生成符合要求的硬盘数据。

### 架构分析

#### vfs

文件系统的最顶层是提供给用户的系统调用；不过系统调用本身的实现很简单，并且在前几个lab里一直都在使用，只是把内核中的接口暴露给用户而已，在此不详细描述，主要分析内核内如何实现文件系统。

sfs文件系统的最顶层是file结构，针对这个结构提供了文件所必要的原语操作接口，包括打开关闭、读写、获取文件信息、复制等等：

```
struct file {
    enum {
        FD_NONE, FD_INIT, FD_OPENED, FD_CLOSED,
    } status;
    bool readable;
    bool writable;
    int fd;
    off_t pos;
    struct inode *node;
    int open_count;
};

// 相关接口
int file_open(char *path, uint32_t open_flags);
int file_close(int fd);
int file_read(int fd, void *base, size_t len, size_t *copied_store);
int file_write(int fd, void *base, size_t len, size_t *copied_store);
int file_seek(int fd, off_t pos, int whence);
int file_fstat(int fd, struct stat *stat);
int file_fsync(int fd);
int file_getdirentry(int fd, struct dirent *dirent);
int file_dup(int fd1, int fd2);
int file_pipe(int fd[]);
int file_mkfifo(const char *name, uint32_t open_flags);
```

虽然这里有file这个结构体，但实际上在接口中用的、即最终交付给上层的只有int类型的编号(fd)而已，这让使用者无需关注文件实现的细节，是一层比较好的封装。

文件结构中包含了文件的状态信息，如使用情况、是否可读写、当前文件内指针位置等。还有最重要的是inode结构：

```
struct inode {
    union {
        struct device __device_info;
        struct sfs_inode __sfs_inode_info;
    } in_info;
    enum {
        inode_type_device_info = 0x1234,
        inode_type_sfs_inode_info,
    } in_type;
    int ref_count;
    int open_count;
    struct fs *in_fs;
    const struct inode_ops *in_ops;
};

struct inode_ops {
    unsigned long vop_magic;
    int (*vop_open)(struct inode *node, uint32_t open_flags);
    int (*vop_close)(struct inode *node);
    int (*vop_read)(struct inode *node, struct iobuf *iob);
    int (*vop_write)(struct inode *node, struct iobuf *iob);
    int (*vop_fstat)(struct inode *node, struct stat *stat);
    int (*vop_fsync)(struct inode *node);
    int (*vop_namefile)(struct inode *node, struct iobuf *iob);
    int (*vop_getdirentry)(struct inode *node, struct iobuf *iob);
    int (*vop_reclaim)(struct inode *node);
    int (*vop_gettype)(struct inode *node, uint32_t *type_store);
    int (*vop_tryseek)(struct inode *node, off_t pos);
    int (*vop_truncate)(struct inode *node, off_t len);
    int (*vop_create)(struct inode *node, const char *name, bool excl, struct inode **node_store);
    int (*vop_lookup)(struct inode *node, char *path, struct inode **node_store);
    int (*vop_ioctl)(struct inode *node, int op, void *data);
};
```

inode结构中指定了其使用的文件系统和类型，区分了是来自设备还是来自磁盘；同时，在`inode_ops`这一结构中以函数指针的形式存储了这一inode适用的操作，这使得不同的inode可以使用不同的底层实现。

这里，可以看到inode的类型有两种，即设备与sfs，我们进行分开介绍。

#### sfs

SFS是ucore中使用的真实的文件系统，这一层中负责文件节点到磁盘位置的映射。`sfs_inode`结构作为vfs直接保存的结构，其定义如下：

```
struct sfs_inode {
    struct sfs_disk_inode *din;                     /* on-disk inode */
    uint32_t ino;                                   /* inode number */
    bool dirty;                                     /* true if inode modified */
    int reclaim_count;                              /* kill inode if it hits zero */
    semaphore_t sem;                                /* semaphore for din */
    list_entry_t inode_link;                        /* entry for linked-list in sfs_fs */
    list_entry_t hash_link;                         /* entry for hash linked-list in sfs_fs */
};
```

除一些标志位外，最重要的是其直接连接到其对应的硬件节点`sfs_disk_inode`上：

```
struct sfs_disk_inode {
    uint32_t size;                                  /* size of the file (in bytes) */
    uint16_t type;                                  /* one of SFS_TYPE_* above */
    uint16_t nlinks;                                /* # of hard links to this file */
    uint32_t blocks;                                /* # of blocks */
    uint32_t direct[SFS_NDIRECT];                   /* direct blocks */
    uint32_t indirect;                              /* indirect blocks */
};
```

硬件节点从磁盘中直接读出，表示着文件在磁盘中的分布情况。根据每个`sfs_inode`对应的`sfs_disk_inode`，就可以把针对文件的操作对应到针对磁盘块的操作。

最后，在`sfs_fs`这个表示SFS整体文件系统的结构中，包含了SFS的全局信息：

```
struct sfs_fs {
    struct sfs_super super;                         /* on-disk superblock */
    struct device *dev;                             /* device mounted on */
    struct bitmap *freemap;                         /* blocks in use are mared 0 */
    bool super_dirty;                               /* true if super/freemap modified */
    void *sfs_buffer;                               /* buffer for non-block aligned io */
    semaphore_t fs_sem;                             /* semaphore for fs */
    semaphore_t io_sem;                             /* semaphore for io */
    semaphore_t mutex_sem;                          /* semaphore for link/unlink and rename */
    list_entry_t inode_list;                        /* inode linked-list */
    list_entry_t *hash_list;                        /* inode hash linked-list */
};
```

这里，指定了系统所对应的底层硬件。因此，它之下还有一个设备层，将由下节阐述。这里的底层设备，只需要在被读入、写回时满足SFS的规定，也完全可以自由更换，只需在mount时指定即可。因此，这使得文件系统更加灵活。

同时，还有与我们文件系统本身实现相关的资料，如存储superblock的信息，用于映射的bitmap，以及dirty位、节点链表等。具体的操作，如加载节点、创建节点、修改节点等过于庞杂，在这里不赘述。

#### 设备

描述设备的结构体`device`的定义如下所示：

```
struct device {
    size_t d_blocks;
    size_t d_blocksize;
    int (*d_open)(struct device *dev, uint32_t open_flags);
    int (*d_close)(struct device *dev);
    int (*d_io)(struct device *dev, struct iobuf *iob, bool write);
    int (*d_ioctl)(struct device *dev, int op, void *data);
};
```

可见，设备层又使用函数指针，对上层（sfs或vfs层）提供了打开、关闭、读写的接口。这个接口共有三个具体实现：磁盘，标准输入，标准输出。在此不分别说明这些设备的细节了。

### 运行分析

接下来，我们以打开一个文件为例，观察每层都进行了什么操作。

调用file\_open时，首先根据打开方式（读/写）设置`struct file`中的标志位，为文件分配一个fd; 然后交给vfs处理，vfs首先从自己的文件系统中查找出要打开的路径对应的inode，然后调用这个inode中的open函数指针指向的函数，将打开文件的操作交付给下层；不过，打开操作就到此为止了，因为磁盘从读写文件并不需要“打开”才能进行操作。

其他操作也是类似地，将整个操作从上向下逐步传递。这样，就完成了一个灵活性强、可扩展性强的文件系统。

到这里，基本上说明了整个文件系统的架构，可以开始做实验了。

## 实验

### 练习一/读文件

要求实现对文件的IO操作。这里要实现的函数是sfs层最底端，和设备之间的交互的一部分。由于文件的存取是连续的、按照大块内存空间进行的，而硬盘的存取是以块为单位的，我们这里需要实现的即是将这种连续空间转为以块为单位，逐块进行处理。

#### 实现细节

我们需要实现的`sfs_io_nolock`函数的定义如下：

```
sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write)
```

其中buf, offset, alenp分别指示了读/写的缓冲区，文件中的偏移量，需要读的长度。

在函数中，又使用了函数指针的方式，让读/写可以被一同处理：具体来说如下，文件内已经为我们实现好了四个函数：

```
int sfs_rblock(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks);
int sfs_wblock(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks);
int sfs_rbuf(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset);
int sfs_wbuf(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset);
```

这些函数调用设备层的接口，分别表示读多块磁盘、写多块磁盘、读单块磁盘内的一段、写单块磁盘内的一段。在这里，我们根据是读还是写，选用其中的两个：

```
if (write) {
    sfs_buf_op = sfs_wbuf, sfs_block_op = sfs_wblock;
}
else {
    sfs_buf_op = sfs_rbuf, sfs_block_op = sfs_rblock;
}
```

于是，读写就被统一化了。接下来，只需要把连续的一长串文件读写拆分成块。

具体的操作如下：

+ 判断要读/写的起始位置是否和块边界对齐。若否，则将对齐前的一小段进行读写；
+ 对要读/写的成块的数据进行读写；
+ 对末尾一段不足一整块的数据进行读写。

最后，需要注意的是，每次调用api前，需要用`sfs_bmap_load_nolock`来获取实际的磁盘块编号，然后将磁盘块号传入`sfs_buf_op, sfs_block_op`完成磁盘操作。

当然，对整个读写段都位于同一块内的情况，以上的步骤不能直接使用。额外判断即可。

综上，这样就完成了读写文件的操作。

#### 问题回答

请在实验报告中给出设计实现“UNIX的PIPE机制”的概要设方案，鼓励给出详细设计方案。

PIPE机制即进程间通信的一种手段。通过pipe系统调用，可以创建管道；管道实际上是两个抽象文件，其中一个只能写，另一个只能读，这样两个线程分别持有其中之一，就可以进行通信。

为了实现之，管道属于特殊的文件系统：既不属于设备，也不属于SFS可以管理的范畴。因此，需要单独作为一种设备提出。另外，管道的功能相对简单，对于文件需要有的大部分接口，管道都不能使用（如seek, fstats等）。基本上，只需要读写、开关这些接口即可。

具体实现如：建立pipe.c/h文件。在文件内实现如下函数：

+ `struct inode *pipe_create_inode(void)`，用于创建管道两端的inode节点，以及对应的`struct pipe`变量；
+ pipe\_write，用于向管道内写入数据，若管道缓冲区已满，则写入进程阻塞（仅有写入侧可用）；
+ pipe\_read，用于从管道内读出数据，若无数据则当前进程阻塞（仅读出侧可用）；
+ pipe\_open，用于打开管道文件；
+ pipe\_close，用于关闭管道文件。

并且，设立好`inode_ops`类型的两个常量`pipe_read_node_ops, pipe_write_node_ops`，分别存入上面实现的函数的函数指针。

然后，由于管道的两侧同属于一个管道，需要有额外的结构来记录这种关系。譬如，可以设置如下的结构体：

```
struct pipe {
    char buf[MAX_PIPE_LEN]; // 缓冲区
    int index; // 当前写入到的位置
    int fd[2]; // 管道两端的fd
    semaphore_t mutex; // 互斥，保证写入读出不被打断
}
```

管道文件的inode中需要有一个指针(in\_info)，指向上面这个管道结构。这样，就可以通过用户指定的fd，查找到对应的inode，再根据inode查找到对应的pipe结构体，就可以完成读写了。

最后，读写时都需注意随时可能发生阻塞。写入缓冲区满无人读出时，读出缓冲区空无人写入时都会发生阻塞。这只需要参考stdin，用schedule函数完成即可。

#### 对答案

本题和答案的实现，看上去比较不同，但是这是由于分块、分情况的细节，经过观察就会发现实际上是等价的。

不过，有一点是本质不同的：虽然提供了`sfs_block_op`这个可以处理多块的接口，但答案却是将每次读/写的块数设为1，用一个循环逐块进行读写。

这可能是因为逐块读写比较安全，因为当读/写到中间某一块如果出错或其他原因而结束，仍可正常返回，且返回的值为当前成功写入/读出的长度。而若调用一次性完成的api就不能这样做了。

### 练习二/载入文件

这里又一次要求实现`load_icode`。但是和之前lab5不同的是，这次用户程序不是直接被放在主磁盘中，而是在文件系统中。另外，这次用户程序运行时还添加了从外部传入的参数，即传统的argc, argv机制。

#### 实现细节

大部分代码都可以直接从lab5的代码照搬过来，包括elf文件的解析，用户栈和trapframe的设置等。在此只描述需要改动的部分。

##### 从文件系统而非内存中读取数据

在lab8中用`load_icode_read`函数简单地封装了一下`sysfile_read`，让接口更加好用。需要改写的是，原本要获得elfhdr, proghdr,只需将结构体指针指向起始地址即可，但现在需要在栈/堆中分配一块内存，从磁盘将相应大小的数据读进该内存里，然后指针指向这片内存的起始地址，这样才能解析elf文件。同样地，为程序段分配页面时也需要把每页的内容读进内存然后分配。

例如读elfhdr:

```
struct elfhdr elf;
load_icode_read(fd, (void *)&elf, sizeof(struct elfhdr), 0);
```

再如页面的分配：

```
while (start < end) {
    if ((page = pgdir_alloc_page(mm->pgdir, la, perm)) == NULL) {
        goto bad_cleanup_mmap;
    }
    off = start - la, size = PGSIZE - off, la += PGSIZE;
    if (end < la) {
        size -= la - end;
    }

    load_icode_read(fd, (void *)(page2kva(page) + off), size, from_offset);
    // 逐页读入

    start += size, from_offset += size;
}
```

这样就完成了从文件系统读入用户程序。

##### 用户程序参数的设置

这里，需要让第一次跳转之后，栈顶恰好指向我们需要的位置：从栈顶开始的栈中分别存放着正确的argc,argv. 仅仅这样还不足够，还需要把参数本身对应的数据也全部放入栈中，否则用户程序无法引用到。

因argv是指向各参数的指针的数组，故我们在如lab5一样建立好栈后，进行这样的分配：

+ 按照顺序压入所有参数(类型均为`void *`)，记录下每个参数的起始地址；
+ 压入和总参数数量相同长度的数组(`void ** argv`)，并将刚刚记录的各参数的起始地址存入数组中；
+ 压入argv(即上一步的参数指针数组)，最后压入参数总数argc.
+ 将此时的栈顶赋给trapframe结构的esp字段，在之后的trapret中让硬件自动将栈顶置于此处，即完成了参数设置。

完成以上步骤后，运行qemu，出现sh界面，并可以通过输入用户程序名运行用户程序，表明实验已经成功。

#### 问题回答

+ 请在实验报告中给出设计实现基于“UNIX的硬链接和软链接机制”的概要设计方案，鼓励给出详细设计方案


##### 接口设计

实际上在代码里已经留出了相应的接口：

```
int vfs_link(char *old_path, char *new_path); // 硬链接
int vfs_symlink(char *old_path, char *new_path); // 软链接
int vfs_readlink(char *path, struct iobuf *iob); // 读软链接
```

不过就仅此而已，这几个接口向上没有被系统调用使用，向下没有被实现，只是设计的框架。不过既然我们需要设计链接的实现，应该需要继承使用这几个接口。

首先，在文件系统层次，应该实现这几个接口：

```
int sysfile_link(const char *path1, const char *path2);         // set a path1's link as path2
int sysfile_symlink(const char *path1, const char *path2);         // set a path1's sym link as path2
int sysfile_readlink(int fd, void *base);               // Read sym link
```

在更上层，还需要设计这几个接口的系统调用，但系统调用的实现是平凡的，不再赘述。

最后，目前底层的实现还不足以实现软硬链接。底层还需要进行极多的更改：现在读入inode时没有根据类型来判断是否为链接；这样链接即使完成了，存回硬盘重启后也无法识别。并且，也没有实现新建路径的功能，也即现在只能读取用户程序而不能存入。我们不妨假设这些功能已经实现了：可以新建路径，并写回磁盘的inode block区。

##### 代码实现

硬链接也即将已有的inode映射到另一个路径上，两个路径共享同一个inode.

为了实现之，只需要在`vfs_link`被调用时，调用更底层的`sfs_link`（也需要我们自己实现之），这个函数负责在文件系统中对应的路径下（虽然我们目前都只在根目录下操作）新增一个`sfs_disk_entry`结构，其`ino`即inode编号和被链接者相同，但名字不同。同时，`sfs_disk_inode`中的`nlink++`，表示多了一个硬链接。将更新后的目录信息存回磁盘，这样就完成了硬链接。

硬链接不需要读取时进行额外的判断，因为是通过共享inode的方式实现的，只需要通过正常方式访问文件就可以了。

软链接不太相同，需要文件系统本身的支持。每一个软链接对应着一个`type=SFS_TYPE_LINK`的inode，标志着它是一符号链接。

软链接涉及两个操作：设置链接，即在源路径创建一个文件，然后往文件中以字符形式存入链接目标源地址。这样就完成了软链接。另一个是读出链接，这即相当于读出整个文本文件。其实现参考read,write等即可。

这样，大致就可以实现软硬链接了。ucore的文件系统还有许多不完善之处，因此这个设计还有缺漏之处，因为底层欠缺的机制（如创建路径、删除路径、文件夹操作）比较多，这里也只能基于ucore当前实现的功能，改动尽量小地实现目标设计。

#### 对答案

和答案相比是相似的。不过答案有一些优点，比如其考虑情况更加全面，每个`load_icode_read`都会判断是否读入成功，若没有成功即刻清除之前分配的内存、结束函数。这一点我没有考虑到。

最后压入参数的部分，感觉答案写的相对粗暴了一点：

```
uintptr_t stacktop = USTACKTOP - (argv_size/sizeof(long)+1)*sizeof(long);
char** uargv=(char **)(stacktop  - argc * sizeof(char *));
```

其中第一句是要实现栈顶地址对long的对齐，第二句是要空出argv数组，但是都用了看上去有点不好看的方式。我个人还是比较喜欢自己实现的方式：

```
cur_stacktop = ROUNDDOWN(cur_stacktop, sizeof(void *));
char** cur_stacktop_as_pointer = (char **) cur_stacktop;
cur_stacktop_as_pointer -= argc + 1;
```

这样看上去可能会比较清楚一些……？但是这只是实现上的细节和个人喜好问题，应该不是很大的问题。

还有，答案和主程序的版本有一些差异，例如lab中的接口是`copy_files`而答案中是`copy_fs`.这应该也不是很大的问题。

### 知识点分析

这个lab涉及了文件系统的各方面知识，包括描述符、inode、vfs等多种实现机制和概念。

不过，显然有很多知识点没有涉及，因为文件系统实际上也只实现了一个不完全的系统，例如路径文件的创建、删除都没有实现。管道机制、软硬链接也只在报告中提到了。文件缓存、磁盘阵列等知识在实验里也没有涉及。