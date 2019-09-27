- [一、 基础知识回顾](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-一、基础知识回顾)
  - [1. 储存金字塔](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.储存金字塔)
  - [2. 虚拟内存](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-2.虚拟内存)
  - [3. 栈和堆](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-3.栈和堆)
  - [4. 堆内存管理](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-4.堆内存管理)
- [二、 TCMalloc](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-二、TCMalloc)
  - [1. 基本原理](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.基本原理)
  - [2. 相关推荐](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-2.相关推荐)
- [三、 GO内存管理](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-三、GO内存管理)
  - [1. Go内存管理的基本概念](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.Go内存管理的基本概念)
    - [1.1 Page](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.1Page)
    - [1.2 Span](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.2Span)
    - [1.3 Mcache](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.3Mcache)
    - [1.4 Mcentral](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.4Mcentral)
    - [1.5 Mheap](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.5Mheap)
    - [1.6 大小转换](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.6大小转换)
  - [2. Go内存分配](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-2.Go内存分配)
    - [2.1 小对象分配](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-2.1小对象分配)
    - [2.2 大对象分配](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-2.2大对象分配)
- [四、 Go的栈内存](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-四、Go的栈内存)
- [五、 代码讲解](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-五、代码讲解)
- [六、 Go垃圾回收和内存释放](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-六、Go垃圾回收和内存释放)
  - [1.1 GC](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.1GC)
  - [1.2 逃逸分析](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-1.2逃逸分析)
- [七、 总结](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-七、总结)
- [八、 参考资料](https://wiki.inkept.cn/pages/viewpage.action?pageId=69317888#go内存分配-八、参考资料)



# 一、 基础知识回顾

## 1. 储存金字塔

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-31-37.png?version=1&modificationDate=1563096697000&api=v2)

这幅图表达了计算机的存储体系，从上至下依次是：

- CPU寄存器
- Cache
- 内存
- 硬盘等辅助存储设备
- 鼠标等外接设备

从上至下，访问速度越来越慢，访问时间越来越长。CPU速度很快，但硬盘等持久存储很慢，如果CPU直接访问磁盘，磁盘可以拉低CPU的速度，机器整体性能就会低下，为了弥补这2个硬件之间的速率差异，所以在CPU和磁盘之间增加了比磁盘快很多的内存。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-33-0.png?version=1&modificationDate=1563096780000&api=v2)

然而，CPU跟内存的速率也不是相同的，从上图可以看到，CPU的速率提高的很快（摩尔定律），然而内存速率增长的很慢，*虽然CPU的速率现在增加的很慢了，但是内存的速率也没增加多少，速率差距很大*，从1980年开始CPU和内存速率差距在不断拉大，为了弥补这2个硬件之间的速率差异，所以在CPU跟内存之间增加了比内存更快的Cache，Cache是内存数据的缓存，可以降低CPU访问内存的时间。

不要以为有了Cache就万事大吉了，CPU的速率还在不断增大，Cache也在不断改变，从最初的1级，到后来的2级，到当代的3级Cache，*（有兴趣看cache历史）*。三级Cache分别是L1、L2、L3，它们的速率是三个不同的层级，L1速率最快，与CPU速率最接近，是RAM速率的100倍，L2速率就降到了RAM的25倍，L3的速率更靠近RAM的速率。

**自顶向下，速率越来越低，访问时间越来越长，从磁盘到CPU寄存器，上一层都可以看做是下一层的缓存。**

## 2. 虚拟内存

虚拟内存是当代操作系统必备的一项重要功能了，它向进程屏蔽了底层了RAM和磁盘，并向进程提供了远超物理内存大小的内存空间。看一下虚拟内存的**分层设计**。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-37-21.png?version=1&modificationDate=1563097041000&api=v2)

上图展示了某进程访问数据，当Cache没有命中的时候，访问虚拟内存获取数据的过程。

访问内存，实际访问的是虚拟内存，虚拟内存通过页表查看，当前要访问的虚拟内存地址，是否已经加载到了物理内存，如果已经在物理内存，则取物理内存数据，如果没有对应的物理内存，则从磁盘加载数据到物理内存，并把物理内存地址和虚拟内存地址更新到页表。

有没有Get到：**物理内存就是磁盘存储缓存层**。

另外，在没有虚拟内存的时代，物理内存对所有进程是共享的，多进程同时访问同一个物理内存存在并发访问问题。**引入虚拟内存后，每个进程都要各自的虚拟内存，内存的并发访问问题的粒度从多进程级别，可以降低到多线程级别**。

## 3. 栈和堆

我们现在从虚拟内存，再进一层，看虚拟内存中的栈和堆，也就是进程对内存的管理。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-38-31.png?version=1&modificationDate=1563097111000&api=v2)

上图展示了一个进程的虚拟内存划分，代码中使用的内存地址都是虚拟内存地址，而不是实际的物理内存地址。栈和堆只是虚拟内存上2块不同功能的内存区域：

- 栈在高地址，从高地址向低地址增长。
- 堆在低地址，从低地址向高地址增长。

**栈和堆相比有这么几个好处**：

1. 栈的内存管理简单，分配比堆上快。
2. 栈的内存不需要回收，而堆需要，无论是主动free，还是被动的垃圾回收，这都需要花费额外的CPU。
3. 栈上的内存有更好的局部性，堆上内存访问就不那么友好了，CPU访问的2块数据可能在不同的页上，CPU访问数据的时间可能就上去了。

## 4. 堆内存管理

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-39-22.png?version=1&modificationDate=1563097162000&api=v2)

我们再进一层，当我们说内存管理的时候，主要是指堆内存的管理，因为栈的内存管理不需要程序去操心。如上图所示主要是3部分：**分配内存块，回收内存块和组织内存块**。

在一个最简单的内存管理中，堆内存最初会是一个完整的大块，即未分配内存，当来申请的时候，就会从未分配内存，分割出一个小内存块(block)，然后用链表把所有内存块连接起来。需要一些信息描述每个内存块的基本信息，比如大小(size)、是否使用中(used)和下一个内存块的地址(next)，内存块实际数据存储在data中。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-40-1.png?version=1&modificationDate=1563097201000&api=v2)

一个内存块包含了3类信息，如下图所示，元数据、用户数据和对齐字段，内存对齐是为了提高访问效率。下图申请5Byte内存的时候，就需要进行内存对齐。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-40-38.png?version=1&modificationDate=1563097239000&api=v2)

释放内存实质是把使用的内存块从链表中取出来，然后标记为未使用，当分配内存块的时候，可以从未使用内存块中有先查找大小相近的内存块，如果找不到，再从未分配的内存中分配内存。

上面这个简单的设计中还没考虑内存碎片的问题，因为随着内存不断的申请和释放，内存上会存在大量的碎片，降低内存的使用率。为了解决内存碎片，可以将2个连续的未使用的内存块合并，减少碎片。

# 二、 TCMalloc

**TCMalloc是Thread Cache Malloc的简称，是Go内存管理的起源**，Go的内存管理是借鉴了TCMalloc，随着Go的迭代，Go的内存管理与TCMalloc不一致地方在不断扩大，但**其主要思想、原理和概念都是和TCMalloc一致的**

在Linux里，其实有不少的内存管理库，比如glibc的ptmalloc，FreeBSD的jemalloc，Google的tcmalloc等等，为何会出现这么多的内存管理库？本质都是**在多线程编程下，追求更高内存管理效率**：更快的分配是主要目的。

那如何更快的分配内存？

我们前面提到：

> 引入虚拟内存后，让内存的并发访问问题的粒度从多进程级别，降低到多线程级别。

这是**更快分配内存的第一个层次**。

同一进程的所有线程共享相同的内存空间，他们申请内存时需要加锁，如果不加锁就存在同一块内存被2个线程同时访问的问题。

TCMalloc的做法是什么呢？**为每个线程预分配一块缓存，线程申请小内存时，可以从缓存分配内存**，这样有2个好处：

1. 为线程预分配缓存需要进行1次系统调用，后续线程申请小内存时，从缓存分配，都是在用户态执行，没有系统调用，**缩短了内存总体的分配和释放时间，这是快速分配内存的第二个层次**。
2. 多个线程同时申请小内存时，从各自的缓存分配，访问的是不同的地址空间，无需加锁，**把内存并发访问的粒度进一步降低了，这是快速分配内存的第三个层次**。

## 1. 基本原理

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-41-45.png?version=1&modificationDate=1563097305000&api=v2)

结合上图，介绍TCMalloc的几个重要概念：

1. **Page**：操作系统对内存管理以页为单位，TCMalloc也是这样，只不过TCMalloc里的Page大小与操作系统里的大小并不一定相等，而是倍数关系。《TCMalloc解密》里称x64下Page大小是8KB。
2. **Span**：一组连续的Page被称为Span，比如可以有2个页大小的Span，也可以有16页大小的Span，Span比Page高一个层级，是为了方便管理一定大小的内存区域，Span是TCMalloc中内存管理的基本单位。
3. **ThreadCache**：每个线程各自的Cache，一个Cache包含多个空闲内存块链表，每个链表连接的都是内存块，同一个链表上内存块的大小是相同的，也可以说按内存块大小，给内存块分了个类，这样可以根据申请的内存大小，快速从合适的链表选择空闲内存块。由于每个线程有自己的ThreadCache，所以ThreadCache访问是无锁的。
4. **CentralCache**：是所有线程共享的缓存，也是保存的空闲内存块链表，链表的数量与ThreadCache中链表数量相同，当ThreadCache内存块不足时，可以从CentralCache取，当ThreadCache内存块多时，可以放回CentralCache。由于CentralCache是共享的，所以它的访问是要加锁的。
5. **PageHeap**：PageHeap是堆内存的抽象，PageHeap存的也是若干链表，链表保存的是Span，当CentralCache没有内存的时，会从PageHeap取，把1个Span拆成若干内存块，添加到对应大小的链表中，当CentralCache内存多的时候，会放回PageHeap。如下图，分别是1页Page的Span链表，2页Page的Span链表等，最后是large span set，这个是用来保存中大对象的。毫无疑问，PageHeap也是要加锁的。


![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-42-26.png?version=1&modificationDate=1563097346000&api=v2)

上文提到了小、中、大对象，Go内存管理中也有类似的概念，我们瞄一眼TCMalloc的定义：

1. 小对象大小：0~256KB
2. 中对象大小：257~1MB
3. 大对象大小：>1MB

小对象的分配流程：ThreadCache -> CentralCache -> HeapPage，大部分时候，ThreadCache缓存都是足够的，不需要去访问CentralCache和HeapPage，无锁分配加无系统调用，分配效率是非常高的。

中对象分配流程：直接在PageHeap中选择适当的大小即可，128 Page的Span所保存的最大内存就是1MB。

大对象分配流程：从large span set选择合适数量的页面组成span，用来存储数据。

## 2. 相关推荐

本文对于TCMalloc的介绍并不多，**重要的是3个快速分配内存的层次**，如果想了解更多，可阅读下面文章。

1. TCMalloc  http://goog-perftools.sourceforge.net/doc/tcmalloc.html
   **必读**，通过这篇你能掌握TCMalloc的原理和性能，对掌握Go的内存管理有非常大的帮助，虽然如今Go的内存管理与TCMalloc已经相差很大，但是，这是**Go内存管理的起源和“大道”**，这篇文章顶看十几篇Go内存管理的文章。
2. TCMalloc解密  https://wallenwang.com/2018/11/tcmalloc/
   **可选**，**异常详细，包含大量精美图片**，看完得花小时级别，理解就需要更多时间了，看完这篇不需要看其他TCMalloc的文章了。
3. TCMalloc介绍
   **可选**，算是TCMalloc的文档的中文版，多数是从英文版翻译过来的，如果你英文不好，看看。

# 三、 GO内存管理

前文提到**Go内存管理源自TCMalloc，但它比TCMalloc还多了2件东西：逃逸分析和垃圾回收**，这是2项提高生产力的绝佳武器。

这一大章节，我们先介绍Go内存管理和Go内存分配，最后涉及一点垃圾回收和内存释放。

## 1. Go内存管理的基本概念

前面计算机基础知识回顾，是一种自上而下，从宏观到微观的介绍方式，把目光引入到今天的主题。

Go内存管理的许多概念在TCMalloc中已经有了，含义是相同的，只是名字有一些变化。先给大家上一幅宏观的图，借助图一起来介绍。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-44-17.png?version=1&modificationDate=1563097457000&api=v2)

### 1.1 Page

- 与TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

### 1.2 Span

- 与TCMalloc中的Span相同，**Span是内存管理的基本单位**，代码中为`mspan`，**一组连续的Page组成1个Span**，所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

### 1.3 Mcache

- mcache与TCMalloc中的ThreadCache类似，**mcache保存的是各种大小的Span，并按Span class分类，小对象直接从mcache分配内存，它起到了缓存的作用，并且可以无锁访问**。
- 但mcache与ThreadCache也有不同点，TCMalloc中是每个线程1个ThreadCache，Go中是**每个P拥有1个mcache**，因为在Go程序中，当前最多有GOMAXPROCS个线程在运行，所以最多需要GOMAXPROCS个mcache就可以保证各线程对mcache的无锁访问，线程的运行又是与P绑定的，把mcache交给P刚刚好

### 1.4 Mcentral

- mcentral与TCMalloc中的CentralCache类似，**是所有线程共享的缓存，需要加锁访问**，它按Span class对Span分类，串联成链表，当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。
- 但mcentral与CentralCache也有不同点，CentralCache是每个级别的Span有1个链表，mcache是每个级别的Span有2个链表，这和mcache申请内存有关，稍后我们再解释。

### 1.5 Mheap

- mheap与TCMalloc中的PageHeap类似，**它是堆内存的抽象，把从OS申请出的内存页组织成Span，并保存起来**。当mcentral的Span不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，同样也是需要加锁访问的。
- 但mheap与PageHeap也有不同点：mheap把Span组织成了树结构，而不是链表，并且还是2棵树，然后把Span分配到heapArena进行管理，它包含地址映射和span是否包含指针等位图，这样做的主要原因是为了更高效的利用内存：分配、回收和再利用。

### 1.6 大小转换

除了以上内存块组织概念，还有几个重要的大小概念，一定要拿出来讲一下，不要忽视他们的重要性，他们是内存分配、组织和地址转换的基础。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-45-53.png?version=1&modificationDate=1563097553000&api=v2)

1. **object size**：代码里简称`size`，指申请内存的对象大小。
2. **size class**：代码里简称`class`，它是size的级别，相当于把size归类到一定大小的区间段，比如size[1,8]属于size class 1，size(8,16]属于size class 2。
3. **span class**：指span的级别，但span class的大小与span的大小并没有正比关系。span class主要用来和size class做对应，1个size class对应2个span class，2个span class的span大小相同，只是功能不同，1个用来存放包含指针的对象，一个用来存放不包含指针的对象，不包含指针对象的Span就无需GC扫描了。
4. **num of page**：代码里简称`npage`，代表Page的数量，其实就是Span包含的页数，用来分配内存。

在介绍这几个大小之间的换算前，我们得先看下图这个表，这个表决定了映射关系。

最上面2行是我手动加的，前3列分别是size class，object size和span size，根据这3列做size、size class和num of page之间的转换。

*另外，第4列num of objects代表是当前size class级别的Span可以保存多少对象数量，第5列tail waste是span%obj计算的结果，因为span的大小并不一定是对象大小的整数倍。最后一列max waste代表最大浪费的内存百分比，计算方法在printComment函数中，没搞清为何这样计算。*

仔细看一遍这个表，再向下看转换是如何实现的。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-47-5.png?version=1&modificationDate=1563097626000&api=v2)

在Go内存大小转换那幅图中已经标记各大小之间的转换，分别是数组：`class_to_size`，`size_to_class*`和`class_to_allocnpages`，这3个数组内容，就是跟上表的映射关系匹配的。比如`class_to_size`，从上表看class 1对应的保存对象大小为8，所以`class_to_size[1]=8`，span大小为8192Byte，即8KB，为1页，所以`class_to_allocnpages[1]=1`。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-47-41.png?version=1&modificationDate=1563097662000&api=v2)

**为何不使用函数计算各种转换，而是写成数组？**

有1个很重要的原因：**空间换时间**。你如果仔细观察了，上表中的转换，并不能通过简单的公式进行转换，比如size和size class的关系，并不是正比的。这些数据是使用较复杂的公式计算出来的，公式在`makesizeclass.go`中，这其中存在指数运算与for循环，造成每次大小转换的时间复杂度为O(N*2^N)。另外，对一个程序而言，内存的申请和管理操作是很多的，如果不能快速完成，就是非常的低效。把以上大小转换写死到数组里，做到了把大小转换的时间复杂度直接降到O(1)。

## 2. Go内存分配

涉及的概念已经讲完了，我们看下Go内存分配原理。

Go中的内存分类并不像TCMalloc那样分成小、中、大对象，但是它的小对象里又细分了一个Tiny对象，Tiny对象指大小在1Byte到16Byte之间并且不包含指针的对象。小对象和大对象只用大小划定，无其他区分。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-48-47.png?version=1&modificationDate=1563097727000&api=v2)

小对象是在mcache中分配的，而大对象是直接从mheap分配的，从小对象的内存分配看起。

### 2.1 小对象分配

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-49-15.png?version=1&modificationDate=1563097755000&api=v2)

大小转换这一小节，我们介绍了转换表，size class从1到66共66个，代码中`_NumSizeClasses=67`代表了实际使用的size class数量，即67个，从0到67，size class 0实际并未使用到。

上文提到1个size class对应2个span class：

```
1numSpanClasses = _NumSizeClasses * 2
```

`numSpanClasses`为span class的数量为134个，所以span class的下标是从0到133，所以上图中mcache标注了的span class是，`span class 0`到`span class 133`。每1个span class都指向1个span，也就是mcache最多有134个span。

-  为对象寻找span

寻找span的流程如下：

1. 计算对象所需内存大小size
2. 根据size到size class映射，计算出所需的size class
3. 根据size class和对象是否包含指针计算出span class
4. 获取该span class指向的span。

以分配一个不包含指针的，大小为24Byte的对象为例。

根据映射表：

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-50-54.png?version=1&modificationDate=1563097855000&api=v2)

size class 3，它的对象大小范围是(16,32]Byte，24Byte刚好在此区间，所以此对象的size class为3。

Size class到span class的计算如下：

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-51-20.png?version=1&modificationDate=1563097880000&api=v2)

```
所以，对应的span class为：

所以该对象需要的是span class 7指向的span。
```

- 从span分配对象空间

Span可以按对象大小切成很多份，这些都可以从映射表上计算出来，以size class 3对应的span为例，span大小是8KB，每个对象实际所占空间为32Byte，这个span就被分成了256块，可以根据span的起始地址计算出每个对象块的内存地址。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-52-55.png?version=1&modificationDate=1563097975000&api=v2)

随着内存的分配，span中的对象内存块，有些被占用，有些未被占用，比如上图，整体代表1个span，蓝色块代表已被占用内存，绿色块代表未被占用内存。

当分配内存时，只要快速找到第一个可用的绿色块，并计算出内存地址即可，如果需要还可以对内存块数据清零。

- span没有空间怎么分配对象

span内的所有内存块都被占用时，没有剩余空间继续分配对象，mcache会向mcentral申请1个span，mcache拿到span后继续分配对象。

- mcentral向mcache提供span

mcentral和mcache一样，都是0~133这134个span class级别，但每个级别都保存了2个span list，即2个span链表：

1. `nonempty`：这个链表里的span，所有span都至少有1个空闲的对象空间。这些span是mcache释放span时加入到该链表的。
2. `empty`：这个链表里的span，所有的span都不确定里面是否有空闲的对象空间。当一个span交给mcache的时候，就会加入到empty链表。

这2个东西名称一直有点绕，建议直接把empty理解为没有对象空间就好了。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-55-12.png?version=1&modificationDate=1563098112000&api=v2)

*实际代码中每1个span class对应1个mcentral，图里把所有mcentral抽象成1个整体了。*

mcache向mcentral要span时，mcentral会先从`nonempty`搜索满足条件的span，如果没找到再从`emtpy`搜索满足条件的span，然后把找到的span交给mcache。

- mheap的span管理

mheap里保存了2棵**二叉排序树**，按span的page数量进行排序：

1. `free`：free中保存的span是空闲并且非垃圾回收的span。
2. `scav`：scav中保存的是空闲并且已经垃圾回收的span。

如果是垃圾回收导致的span释放，span会被加入到`scav`，否则加入到`free`，比如刚从OS申请的的内存也组成的Span。

![img](https://wiki.inkept.cn/download/attachments/69317888/image2019-7-14_17-56-48.png?version=1&modificationDate=1563098208000&api=v2)

mheap中还有arenas，有一组heapArena组成，每一个heapArena都包含了连续的`pagesPerArena`个span，这个主要是为mheap管理span和垃圾回收服务。

mheap本身是一个全局变量，它其中的数据，也都是从OS直接申请来的内存，并不在mheap所管理的那部分内存内。

- mcentral向mheap要span

mcentral向mcache提供span时，如果`emtpy`里也没有符合条件的span，mcentral会向mheap申请span。

mcentral需要向mheap提供需要的内存页数和span class级别，然后它优先从`free`中搜索可用的span，如果没有找到，会从`scav`中搜索可用的span，如果还没有找到，它会向OS申请内存，再重新搜索2棵树，必然能找到span。如果找到的span比需求的span大，则把span进行分割成2个span，其中1个刚好是需求大小，把剩下的span再加入到`free`中去，然后设置需求span的基本信息，然后交给mcentral。

- mheap向OS申请内存

当mheap没有足够的内存时，mheap会向OS申请内存，把申请的内存页保存到span，然后把span插入到`free`树 。

在32位系统上，mheap还会预留一部分空间，当mheap没有空间时，先从预留空间申请，如果预留空间内存也没有了，才向OS申请。

### 2.2 大对象分配

大对象的分配比小对象省事多了，99%的流程与mcentral向mheap申请内存的相同，所以不重复介绍了，不同的一点在于mheap会记录一点大对象的统计信息，见`mheap.alloc_m()`。

# 四、 Go的栈内存

最后提一下栈内存。从一个宏观的角度看，内存管理不应当只有堆，也应当有栈。

每个goroutine都有自己的栈，栈的初始大小是2KB，100万的goroutine会占用2G，但goroutine的栈会在2KB不够用时自动扩容，当扩容为4KB的时候，百万goroutine会占用4GB。

关于goroutine栈内存管理，有篇很好的文章，饿了么框架技术部的专栏文章：《聊一聊goroutine stack》，把里面的一段内容摘录下：

> 可以看到在rpc调用(*grpc invoke*)时，栈会发生扩容(*runtime.morestack*)，也就意味着在读写routine内的任何rpc调用都会导致栈扩容， 占用的内存空间会扩大为原来的两倍，4kB的栈会变为8kB，100w的连接的内存占用会从8G扩大为16G（全双工，不考虑其他开销），这简直是噩梦。

# 五、 代码讲解

- new一个对象的时候，入口函数是malloc.go中的newobject函数

 展开源码

- 这个函数先计算出传入参数的大小，然后调用mallocgc函数，这个函数三个参数，第一个参数是对象类型大小，第二个参数是对象类型，第三个参数是malloc的标志位，这个标志位有两位，一个标志位代表GC不需要扫描这个对象，另一个标志位说明这个对象并不是空内存

 展开源码

总结一下

- 如果要申请的对象是tiny大小，看mcache中的tiny block是否足够，如果足够，直接分配。如果不足够，使用mcache中的tiny class对应的span分配
- 如果要申请的对象是小对象大小，则使用mcache中的对应span链表分配
- 如果对应span链表已经没有空span了，先补充上mcache的对应链表，再分配（mCache_Refill）
- 如果要申请的对象是大对象，直接去heap中获取（largeAlloc）

再仔细看代码，不管是tiny大小的对象还是小对象，他们去mcache中获取对象都是使用mCache_Refill方法为这个对象对应的链表申请内存。

 展开源码


线程从central获取span步骤如下：

1. 加锁
2. 从nonempty列表获取一个可用span，并将其从链表中删除
3. 将取出的span放入empty链表
4. 将span返回给线程
5. 解锁
6. 线程将该span缓存进cache

线程将span归还步骤如下：

1. 加锁
2. 将span从empty列表删除
3. 将span加入nonempty列表
4. 解锁



# 六、 Go垃圾回收和内存释放

### 1.1 GC

如果只申请和分配内存，内存终将枯竭，Go使用垃圾回收收集不再使用的span，调用`mspan.scavenge()`把span释放给OS（并非真释放，只是告诉OS这片内存的信息无用了，如果你需要的话，收回去好了），然后交给mheap，mheap对span进行span的合并，把合并后的span加入`scav`树中，等待再分配内存时，由mheap进行内存再分配，Go垃圾回收也是一个很强的主题。

### 1.2 逃逸分析

- 什么是逃逸分析

在编译程序优化理论中，逃逸分析是一种确定指针动态范围的方法，简单来说就是分析在程序的哪些地方可以访问到该指针。

再往简单的说，Go是通过在编译器里做逃逸分析（escape analysis）来决定一个对象放栈上还是放堆上，不逃逸的对象放栈上，可能逃逸的放堆上；即我发现`变量`在退出函数后没有用了，那么就把丢到栈上，毕竟栈上的内存分配和回收比堆上快很多；反之，函数内的普通变量经过`逃逸分析`后，发现在函数退出后`变量`还有在其他地方上引用，那就将`变量`分配在堆上，做到按需分配。

- 为何需要逃逸分析

1. 减少`gc`压力，栈上的变量，随着函数退出后系统直接回收，不需要`gc`标记后再清除。
2. 减少内存碎片的产生。
3. 减轻分配堆内存的开销，提高程序的运行速度。

# 七、 总结

内存分配原理强调2个重要的思想：

1. **使用缓存提高效率**。在存储的整个体系中到处可见缓存的思想，Go内存分配和管理也使用了缓存，利用缓存一是减少了系统调用的次数，二是降低了锁的粒度，减少加锁的次数，从这2点提高了内存管理效率。
2. **以空间换时间，提高内存管理效率**。空间换时间是一种常用的性能优化思想，这种思想其实非常普遍，比如Hash、Map、二叉排序树等数据结构的本质就是空间换时间，在数据库中也很常见，比如数据库索引、索引视图和数据缓存等，再如Redis等缓存数据库也是空间换时间的思想。

# 八、 参考资料

1. 全成的内存分配文章：https://juejin.im/post/5c888a79e51d456ed11955a8#heading-5
2. 异常详细的源码分析文章：https://www.cnblogs.com/zkweb/p/7880099.html
3. 从硬件讲起的一篇文章：https://www.infoq.cn/article/IEhRLwmmIM7-11RYaLHR
4. 这篇文章的总流程图很棒：http://media.newbmiao.com/blog/malloc.png