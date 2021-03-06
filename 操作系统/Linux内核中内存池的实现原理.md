# Linux内核中内存池的实现原理

## 1、为什么有内存池

在Linux操作系统总，内核将物理页作为内存管理的基本单位，内存管理单元通常以页为单位进行处理。通常我们习惯直接使用new, malloc等API函数申请分配内存。但在申请内存时，由于所申请内存块的大小不定，当频繁使用这些API函数时会造成大量的内存碎片从而降低性能。

因此Linux2.6中内核引入了内存池。



## 2、Linux系统内存管理

Linux内粗管理中简化了分段机制，使得虚地址与线性地址总是一致的，线性空间在32位平台上为4GB的固定大小。Linux内核这4GB的空间分为两部分，其中0到3GB是用户态空间，由各进程独占；3GB到4GB是内核态空间，所有进程共享，但只有内核态进程才能访问。

Linux内存管理采用了伙伴算法，伙伴算法分配内存时，每次至少分配一个页面。这种方法适合大块内存请求，不适合小内存区请求。当请求分配的内存大小为几十个字节时需要例外的管理机制。

从Linux2.2开始，内存分配时采用了一种新的内存分配器：slab分配器。

![1586691978660](C:\Users\wonde\AppData\Roaming\Typora\typora-user-images\1586691978660.png)

在slab分配器中，最高层cache_chain是一个slab缓存的链接列表，这对于最佳适配算法非常有用，可以用来查找大小最适合的缓存（遍历列表）。cache_chain的每个元素都是一个kmem_cache结构的引用（称为一个cache）。它定义了一个要管理的给定大小的对象池。

与传统的管理模式相比，slab缓存分配器有很多优点。首先，内核通常依赖于对小对象的分配，它们会在系统生命周期内进行无数次分配。slab缓存分配器通过对类似大小的对象进行缓存而提供这种功能，从而避免了常见的碎片问题。

其次，slab分配器还支持通用对象的初始化，从而避免了为同一目标而对一个对象重复进行初始化。

最后，slab分配器还可以支持硬件缓存对齐，这允许不同缓存中的对象占用相同的缓存行，从而提高缓存的利用率并获得更好的性能。



## 3、Linux内存池

内存池是动态分配内存的设备，只能被特定的内核成分（即池的”拥有者“）使用。拥有者通常不直接使用内存池，当普通内存分配失败时，内核才调用特定的内存池函数来提取内存池，以得到所需的额外内存。因此内存池知识内核内存的一个储备，用在特定的时刻。这样做的一个显著优点是尽量避免内存碎片，提高内存分配效率。

一个内存池通常叠加在slab分配器之上——也就是说，它被用来保存slab对象的储备。但是一般而言，内存池能被用来分配任何一种类型的动态内存，从整个页框到使用kmalloc函数分配的小内存区。因此，我们可以将一般内存池处理的内存单元看成”内存元素“。

引入内存池之后，Linux内核的内存管理采取两级分配机制，即初始化时将内存池按照内存的大小分成多个级别（每个级别均是8字节的整数倍，一般是8，16，24，。。。128字节），每个级别都预先分配了20块内存。如果用户申请的内存单元大于预定义的界别，则直接调用malloc从堆中分配内存。而如果申请的内存小于128字节，则从最相近的内存大小中申请，如果该组的内存存储量小于一定的值，就会根据算法，再次从堆中申请一部分内存加入内存池，保证内存池中有一定量的内存你可以使用。



## 4、内存池的数据结构

Linux内核中内存池的数据结构定义在<linux/mempool.h>中。

内存池mempool_s定义为：

```c++
typedef struct mempool_s{
    spinlock_t lock;
    int min_nr;
    void * * elements;
    void * pool_data;
    mempool_alloc_t * alloc;
    mempool_free_t * free;
    wait_queue_head_t wait;
}memepool_t;
```

结构体中的相应字段：

lock: 用来保护对象字段的自旋锁；

min_nr: 内存池中元素的最小个数，elements数组中空闲的成员数量；

curr_nr: 当前内存池中元素的个数，即当前elements数组中空闲的成员数量；

elements: 用来存放内存成员的二维数组，其长度为min_nr, 宽度是上述各个内存对象的长度，因为对于不同的对象类型，会建立相应的内存池对象，所以每个内存池对象实例的宽度都是跟其内存对象有关的；

pool_data: 内存池与内存缓冲区的结合使用，这个指针通常是指向这种内存对象对应的缓存区的指针。由于Linux采用slab技术预先为每一种内存对象分配了缓存区，每当我们申请某个类型的内存对象时，实际是从这种缓存区获取内存。

alloc: 用户在创建一个内存池对象时提供的内存分配函数，这个函数可以用户编写，也可以采用内置函数。

free: 与alloc功能相反的一个函数，内存释放函数。

wait: 当内存池为空时使用的等待队列。



## 5、进程如何使用内存

毫无疑问，所有进程（执行的程序）都必须占用一定数量的内存，它或是用来存放从磁盘载入的程序代码，或是存放取自用户输入的数据等等。不过进程对这些内存的管理方式因内存用途不一而不尽相同，有些内存是事先静态分配和统一回收的，而有些却是按需要动态分配和回收的。

对任何一个普通进程来讲，它都会涉及到5种不同的数据段。稍有编程知识的朋友都能想到这几个数据段中包含有“程序代码段”、“程序数据段”、“程序堆栈段”等。不错，这几种数据段都在其中，但除了以上几种数据段之外，进程还另外包含两种数据段。下面我们来简单归纳一下进程对应的内存空间中所包含的5种不同的数据区。

**代码段：**代码段是用来存放可执行文件的操作指令，也就是说是它是可执行程序在内存中的镜像。代码段需要防止在运行时被非法修改，所以只准许读取操作，而不允许写入（修改）操作——它是不可写的。

**数据段：**数据段用来存放可执行文件中已初始化全局变量，换句话说就是存放程序静态分配[1]的变量和全局变量。

**BSS段[2]：**BSS段包含了程序中未初始化的全局变量，在内存中 bss段全部置零。

**堆（heap）：**堆是用于存放进程运行中被动态分配的内存段，它的大小并不固定，可动态扩张或缩减。当进程调用malloc等函数分配内存时，新分配的内存就被动态添加到堆上（堆被扩张）；当利用free等函数释放内存时，被释放的内存从堆中被剔除（堆被缩减）

**栈：**栈是用户存放程序临时创建的局部变量，也就是说我们函数括弧“{}”中定义的变量（但不包括static声明的变量，static意味着在数据段中存放变量）。除此以外，在函数被调用时，其参数也会被压入发起调用的进程栈中，并且待到调用结束后，函数的返回值也会被存放回栈中。由于栈的先进先出特点，所以栈特别方便用来保存/恢复调用现场。从这个意义上讲，我们可以把堆栈看成一个寄存、交换临时数据的内存区。

![img](https://img-blog.csdn.net/20140904215636015?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemhhbmd6aGVianV0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## 参考

王小银, 陈莉君. Linux内核中内存池的实现及应用[J]. 西安邮电学院学报, 2011(04):46-49.

https://blog.csdn.net/to_be_better/article/details/55250397

https://blog.csdn.net/yusiguyuan/article/details/45155035