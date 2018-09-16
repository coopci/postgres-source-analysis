## 第二章  共享内存 ##

shmem.c 文件开头的注释对共享内存的使用方法有个大致的说明，这段说明已经把pg中使用共享内存的大原则解释得很明确。
本章主要分析一些比较重要的细节并对这段注释中过于概括的部分加以补充。
每一个共享的数据结构都有一个字符串名字，例如 叫做"Buffer Blocks"的共享数据结构是用来维护数据库表的页对应的内存buffer；叫做"Buffer Descriptors"的共享数据结构是对"Buffer Blocks"维护的页buffer的描述，至于描述的内容会在后面进行分析。
shmem.c 中的定义的两个函数 ShmemInitStruct 和 ShmemInitHash 是用来从共享内存区域分配共享数据结构的（ShmemInitHash要调用ShmemInitStruct）。
它们的第一个参数const char *name 就表示上面说到的共享数据结构的名字。
所以只要找到ShmemInitStruct 和 ShmemInitHash被调用的地方就能知道postgres里面一共有哪些共享数据结构，而且能从名字大概猜测这个数据结构的目的何在。
分配得到的数据结构的地址会被存到各自的全局变量里，这些全局标量的名字和数据结构的字符串名字基本一致，例如"Buffer Descriptors"结构会被存到shmem.c文件的BufferDescPadded *BufferDescriptors，"Buffer Strategy Status" 会被存到freelist.c文件的 static BufferStrategyControl *StrategyControl。
这样主进程初始化这些全局变量和共享数据结构的关系以后，fork(在第一章已经知道pg要为每一个客户端连接 fork一个专用的子进程)出的子进程会继承相同的对应关系。

其中一个叫做的"ShmemIndex"的共享数据结构比较特殊，它的作用是维护共享数据结构名字和数据结构地址的对应关系。除了全局变量ShmemIndex之外，"ShmemIndex"的地址还会存在ShmemSegHdr->index里。

整个共享内存区域由这全局变量 static PGShmemHeader *ShmemSegHdr 维护。 
PGSharedMemoryCreate 函数调用系统调用分配整个共享内存区域，这个函数是操作系统相关的，所以针对不同操作有多个版本。 
用 PGSharedMemoryCreate 分配得到的整个内存区域会由InitShmemAccess关联到对应的全局变量上面。

"ShmemIndex" 的初始化由InitShmemIndex函数开始:
```
ShmemIndex = ShmemInitHash("ShmemIndex",
							   SHMEM_INDEX_SIZE, SHMEM_INDEX_SIZE,
							   &info, hash_flags);
```

最终也在ShmemInitStruct里面:
```
structPtr = ShmemAlloc(size); 
shmemseghdr->index = structPtr; 
```

ShmemAlloc 的功能是从整个共享内存区域分配一块连续的内存。
总结一下 ShmemInitHash，ShmemInitStruct 和 ShmemAlloc 三者的关系：
ShmemInitStruct 会根据name去ShmemIndex寻找当前要分配的数据结构是否已经存在，如果存在则直接返回，否则用ShmemAlloc分配一块指定大小的连续空间并注册到ShmemIndex。如果ShmemIndex尚未初始化，则会进行初始化ShmemIndex的特殊流程。
ShmemInitHash 会调用ShmemInitStruct来分配或者attach到已经分配的数据结构上。并在分配结果上创建hash table结构（通过调用hash_create）。后面会有专门分析pg中hash table的章节。 



本章对pg中使用共享内存的方法做了大致的说明，下一章我们结合共享内存中的"Buffer Descriptors"结构和"Buffer Blocks"结构，对第一章中提到的seqscan做更细致的分析。




