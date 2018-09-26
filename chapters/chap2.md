## 第二章  共享内存，页buffer 和 SeqScan ##
因为pg服务器要为每一个客户端连接fork一个专门对其进行服务的子进程，所以显然地，很多数据需要在这些子进程之间共享(还有一些后台任务的子进程也要共享这些数据，不过我们之后再考虑这些后台进程)，例如其中占空间最多的共享数据结构就是数据文件的内容的buffer。

pg的数据文件和索引文件都被划分成固定大小的页(典型的页大小8KB)，内存和文件系统同步数据以页为单位进行，共享内存的组织和查找同样以页为单位进行 —— 当pg需要读取某条数据时，要先确保该条数据所在的页已经被读到了共享内存中，然后对共享内存做相应的访问；当pg需要写某条数据时，会先写到此数据所在的页在共享内存中的副本上，之后由后台进程把共享内存中的副本写到文件系统中。

本章要分析的就是pg组织共享内存的方法，以及页buffer作为共享内存的一部分是如何被SeqScan的。SeqScan是逻辑上最简单的一种搜索数据的执行策略，它要做的是检查表中的每一个row，把符合筛选条件的row输出。综合上面刚说的"表中的数据都是按页存放"，那么我们不难推测出SeqScan的执行会是把表的每个页按顺序读到共享内存，然后对共享内存的页按顺序逐row筛选。

多进程共享相同的数据区域，自然要考虑同步的问题，不过在这一章我们先把关注点放在数据操作上面，后面用单独的章节分析pg中同步多个子进程的手段。

### 2.1 共享内存

shmem.c 文件开头的注释对共享内存的使用方法有个大致的说明，这段说明已经把pg中使用共享内存的大原则解释得很明确。
从设计上说有三个重要原则:
1. 一共有三种共享数据结构，一是固定大小的一片内存。二是最大大小固定的hash table。三是用来链接前两者的队列结构。 
2. 有一个特殊的共享数据结构叫做 "Shmem Index"，pg用它来维护其他共享数据结构的名字和在共享内存中的位置。
3. 共享内存一旦分配就不会释放了。对于hash table 这种越用越大的结构，删掉的表项的空间只会被重用不会被释放。

本节主要分析一些比较重要的细节并对这段注释中过于概括的部分加以补充。
每一个共享的数据结构都有一个字符串名字，例如 叫做"Buffer Blocks"的共享数据结构是用来维护数据库表的页对应的内存buffer；叫做"Buffer Descriptors"的共享数据结构是对"Buffer Blocks"维护的页buffer的描述，至于描述的内容会在后面进行分析。
shmem.c 中定义的两个函数 ShmemInitStruct 和 ShmemInitHash 是用来从整个共享内存区域分配共享数据结构的（ShmemInitHash要调用ShmemInitStruct）。
它们的第一个参数const char *name 就表示上面说到的共享数据结构的名字。
所以只要找到ShmemInitStruct 和 ShmemInitHash被调用的地方就能知道postgres里面一共有哪些共享数据结构，而且能从名字大概猜测这个数据结构的目的何在。
分配得到的数据结构的地址会被存到各自的全局变量里，这些全局标量的名字和数据结构的字符串名字基本一致，例如"Buffer Descriptors"结构会被存到shmem.c文件的BufferDescPadded *BufferDescriptors，"Buffer Strategy Status" 会被存到freelist.c文件的 static BufferStrategyControl *StrategyControl。
这样主进程初始化这些全局变量和共享数据结构的关系以后，fork(在第一章已经知道pg要为每一个客户端连接 fork一个专用的子进程)出的子进程会继承相同的对应关系。在继承上面EXEC_BACKEND是个例外，需要利用"Shmem Index"重新初始全局变量。

对于特殊的"ShmemIndex"，除了全局变量ShmemIndex之外，"ShmemIndex"的地址还会被存到ShmemSegHdr->index里。

整个共享内存区域由这全局变量 static PGShmemHeader *ShmemSegHdr 维护。 pg是在主进程启动时就把整个共享内存区域分配好，之后都是从这个一次性分配好的区域中一块一块地分配内存给不同的共享数据结构。各个处理客户端请求的子进程需要attach到由主进程分配的共享区域以及各种共享数据结构。 

这个区域总大小的计算和分配都在CreateSharedMemoryAndSemaphores函数中。在CreateSharedMemoryAndSemaphores里面，PGSharedMemoryCreate 函数调用系统调用按计算好的总大小分配整个共享内存区域，PGSharedMemoryCreate是操作系统相关的，所以针对不同操作有多个版本。 用 PGSharedMemoryCreate 分配得到的整个内存区域会由InitShmemAccess关联到全局变量static PGShmemHeader *ShmemSegHdr上面。

在完成ShmemSegHdr的初始化之后，就要建立"ShmemIndex"，"ShmemIndex" 的初始化由InitShmemIndex函数开始:
```
ShmemIndex = ShmemInitHash("ShmemIndex",
							   SHMEM_INDEX_SIZE, SHMEM_INDEX_SIZE,
							   &info, hash_flags);
```

最终也在ShmemInitStruct里面，ShmemInitStruct需要用到已经初始化好的全局变量static PGShmemHeader *ShmemSegHdr:
```
PGShmemHeader *shmemseghdr = ShmemSegHdr;
...
structPtr = ShmemAlloc(size); 
shmemseghdr->index = structPtr; 
```

ShmemAlloc 的功能是从整个共享内存区域分配一块连续的内存。
总结一下 ShmemInitHash，ShmemInitStruct 和 ShmemAlloc 三者的关系：
ShmemInitStruct 会根据name去ShmemIndex寻找当前要分配的数据结构是否已经存在，如果存在则直接返回，否则用ShmemAlloc分配一块指定大小的连续空间并注册到ShmemIndex。如果ShmemIndex尚未初始化，则会进行初始化ShmemIndex的特殊流程。
ShmemInitHash 会调用ShmemInitStruct来分配或者attach到已经分配的数据结构上。并在分配结果上创建hash table结构（通过调用hash_create）。后面会有专门分析pg中hash table的章节。 

#### EXEC_BACKEND模式
在EXEC_BACKEND(编译选项)这种模式下子，进程不直接继承这些存放共享内存的全局变量，所以pg的代码要在子进程中建立和主进程相同的共享数据结构。在windows平台上的实现做法大致是这样: 将共享内存的句柄(handle)和共享内区域的地址分别保存在全局变量UsedShmemSegID和UsedShmemSegAddr中。主进程为了向子进程传递这些信息(在BackendParameters结构中定义了所有需要通过这种手段共享的信息，包含但不只UsedShmemSegID和UsedShmemSegAddr)，要单独建立一个临时的共享内区域，通过save_backend_variables函数把BackendParameters的内容拷贝到临时共享区域中，并把临时共享区域的句柄以ascii的形式通过fork命令行参数穿给子进程。在子进程中恢复信息的流程如下:
```
main.c:main
    postgres.c:SubPostmasterMain 
        postgres.c:read_backend_variables   从命令参数影射临时共享区域到局部变量BackendParameters param，并以param为参数调用restore_backend_variables。
            postgres.c:restore_backend_variables   从param恢复 UsedShmemSegID，UsedShmemSegAddr 等全局变量。
        postgres.c:PGSharedMemoryReAttach    以restore_backend_variables恢复得到的 UsedShmemSegID，UsedShmemSegAddr 作为参数将主共享区域影射到本进程的地址空间。
        shmem.c:InitShmemAccess  用UsedShmemSegAddr恢复ShmemSegHdr，ShmemBase和ShmemEnd。
        ipci.c:CreateSharedMemoryAndSemaphores  把表示各个共享数据结构的全局变量attach到从ShmemSegHdr指出的主共享区的相应位置。

```

### 2.2 页buffer
#### BLCKSZ，shared_buffers和NBuffers
有三个数字用来确定页buffer的布局:
    1. BLCKSZ，这是个宏定义，用来指定一页的大小，默认是8192字节。可以在编译pg的时候的自定义。
    2. shared_buffers， 这个是 postgresql.conf里面的一个配置项，用来告诉pg我们希望总共用多少共享内存来作为页buffer。可以在启动pg主进程之前指定，如果要改的话需要重启pg主进程。
    3. NBuffers = shared_buffers / BLCKSZ。 代码中的一个全局变量，记录内存中同时最多可以把多少个页放到buffer里面。
#### 与页buffer相关的共享数据结构
pg里面用三个共享数据结构来维护数据文件的页在内存中的影射，分别是:
1. BufferBlocks "Buffer Blocks"               长度固定是NBuffers的数组，每一项都是一个数据页(长度是BLCKSZ字节)。每个页内部的布局在官方文档66.6Database Page Layout有详细描述。
2. BufferDescriptors "Buffer Descriptors"     长度固定是NBuffers的数组，数组元素的类型是 BufferDescPadded。 这个数组中的元素和BufferBlocks中的元素是一一对应的，也就是说BufferDescriptors[i]是对BufferBlocks[i]的描述。
3. SharedBufHash   "Shared Buffer Lookup Table"  最多有 NBuffers + NUM_BUFFER_PARTITIONS(128) 个key 的 hash table,  key的类型是BufferTag，value的类型是BufferLookupEnt。这个hash table 用来记录特定的页在上述两个数组中的位置。可以认为BufferTag指明一个页在文件系统中的位置，BufferLookupEnt.id则指出了该页在内存buffer中的位置。
```
// SharedBufHash 值的类型
typedef struct
{
	BufferTag	key;			/* Tag of a disk page */
	int			id;				/* Associated buffer ID */  // 也就是BufferBlocks和BufferDescriptors数组的下标。
} BufferLookupEnt;
```

另有两个辅助的共享数据结构:
1. BufferIOLWLockArray  "Buffer IO Locks"  长度固定是NBuffers的数组，每一项都是一个专门用来同步对buffer做io的锁。
2. CkptBufferIds        "Checkpoint BufferIds"   长度固定是NBuffers的数组，用来在checkpoint时给buffer排序。

##### BufferDescriptors
BufferDescriptors元素内部的内存布局又下面两个结构定义。
```
typedef struct BufferDesc
{
	BufferTag	tag;			/* ID of page contained in buffer */
	int			buf_id;			/* buffer's index number (from 0) */

	/* state of the tag, containing flags, refcount and usagecount */
	pg_atomic_uint32 state;          // 高10位是10个标志位，其中的 BM_LOCKED 位用来同步对 tag, state, wait_backend_pid的访问。 

	int			wait_backend_pid;	/* backend PID of pin-count waiter */
	int			freeNext;		/* link in freelist chain */    // 用StrategyControl->buffer_strategy_lock同步，而不需要BM_LOCKED。

	LWLock		content_lock;	/* to lock access to buffer contents */  // 用来同步对对应的BufferBlocks的访问。
} BufferDesc;

typedef union BufferDescPadded
{
	BufferDesc	bufferdesc;
	char		pad[BUFFERDESC_PAD_TO_SIZE];
} BufferDescPadded;

```
其中state的高10位是10个标志位:
1. BM_LOCKED(1U << 22) 位用来同步对 tag, state, wait_backend_pid的访问。
2. BM_DIRTY(1U << 23)	/* data needs writing */
3. BM_VALID				(1U << 24)	文件系统中的页已经读到和本BufferDesc对应的BufferBlocks元素中了。
4. BM_TAG_VALID			(1U << 25)	tag字段已经赋值过了。
5. BM_IO_IN_PROGRESS		(1U << 26)	正在读入内存或者写入文件系统。
6. BM_IO_ERROR				(1U << 27)	上一次I/O操所失败了。
7. BM_JUST_DIRTIED			(1U << 28)	/* dirtied since write started */
8. BM_PIN_COUNT_WAITER		(1U << 29)	/* have waiter for sole pin */
9. BM_CHECKPOINT_NEEDED	    (1U << 30)	/* must write for checkpoint */
10. BM_PERMANENT			(1U << 31)	/* permanent buffer (not unlogged,  or init fork) */
                            

次高4位是usage_count用于clock sweeps算法寻找可以被evict的页。低18位是refcount用来记录。
###### BM_LOCKED 和 spin+cas

##### SharedBufHash
这个hash table 的key数量比较大，而且访问频繁，pg对此作了一个优化——把它的key空间分成NUM_BUFFER_PARTITIONS(宏定义，默认128)个partition。当需要对某一个key(BufferTag)进行操作时，pg的上锁粒度都是该key所属的partition(计算所属partition的宏是BufTableHashPartition)。



### 2.3 SeqScan


