## 第二章  共享内存，页buffer 和 SeqScan ##
因为pg服务器要为每一个客户端连接fork一个专门对其进行服务的子进程，所以显然地，很多数据需要在这些子进程之间共享(还有一些后台任务的子进程也要共享这些数据，不过我们之后再考虑这些后台进程)，例如其中占空间最多的共享数据结构就是数据文件的内容的buffer。

pg的数据文件和索引文件都被划分成固定大小的页(典型的页大小8KB)，pg以页为单位在内存和文件系统之间同步数据，共享buffer的组织和查找同样以页为单位进行 —— 当pg需要读取某条数据时，要先确保该条数据所在的页已经被读到了共享内存中，然后对共享内存做相应的访问；当pg需要写某条数据时，会先写到此数据所在的页在共享内存中的副本上，之后把该页在共享内存中的副本整页写到文件系统中。

本章要分析的就是pg组织共享内存的方法，以及页buffer作为共享内存的一部分是如何被SeqScan的。SeqScan是逻辑上最简单的一种搜索数据的执行策略，它要做的是检查表中的每一个row，把符合筛选条件的row输出。综合上面刚说的"表中的数据都是按页存放"，那么我们不难推测出SeqScan的执行会是把表的每个页按顺序读到共享内存，然后对共享内存中的每个页按顺序逐row筛选。

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
        postgres.c:read_backend_variables   从命令参数映射临时共享区域到局部变量BackendParameters param，并以param为参数调用restore_backend_variables。
            postgres.c:restore_backend_variables   从param恢复 UsedShmemSegID，UsedShmemSegAddr 等全局变量。
        postgres.c:PGSharedMemoryReAttach    以restore_backend_variables恢复得到的 UsedShmemSegID，UsedShmemSegAddr 作为参数将主共享区域影射到本进程的地址空间 —— 需要本进程的UsedShmemSegAddr映射到地址和恢复出的UsedShmemSegAddr相同才行。
        shmem.c:InitShmemAccess  用UsedShmemSegAddr恢复ShmemSegHdr，ShmemBase和ShmemEnd。
        ipci.c:CreateSharedMemoryAndSemaphores  把表示各个共享数据结构的全局变量attach到从ShmemSegHdr指出的主共享区的相应位置。在这种情况下，ShmemIndex是用 ShmemSegHdr->index 恢复得到的。

```

### 2.2 页buffer
#### BLCKSZ，shared_buffers和NBuffers
有三个数字用来确定页buffer的布局:
    1. BLCKSZ，这是个宏定义，用来指定一页的大小，默认是8192字节。
    2. shared_buffers， 这是 postgresql.conf里面的一个配置项，用来告诉pg我们希望总共用多少共享内存来作为页buffer。可以在启动pg主进程之前指定，如果要改的话需要重启pg主进程。
    3. NBuffers = shared_buffers / BLCKSZ。 代码中的一个全局变量，记录内存中同时最多可以把多少个页放到buffer里面。
#### 与页buffer相关的共享数据结构
pg里面用三个共享数据结构来维护数据文件的页在内存中的影射，分别是:
1. BufferBlocks "Buffer Blocks"               长度固定是NBuffers的数组，每一项都是一个数据页(长度是BLCKSZ字节，和文件系统中的每个页一一对应)。每个页内部的布局在官方文档66.6Database Page Layout有详细描述。
2. BufferDescriptors "Buffer Descriptors"     长度固定是NBuffers的数组，数组元素的类型是 BufferDescPadded。 这个数组中的元素和BufferBlocks中的元素是一一对应的，也就是说BufferDescriptors[i]是对BufferBlocks[i]的描述。这个共享结构不会反映到文件系统，是专门用来在pg进程间协调(例如锁，引用计数)对BufferBlocks的访问用的。
3. SharedBufHash   "Shared Buffer Lookup Table"  最多有 NBuffers + NUM_BUFFER_PARTITIONS(128) 个key 的 hash table,  key的类型是BufferTag，value的类型是BufferLookupEnt。这个hash table 用来记录特定的文件系统页在上述两个数组中的位置 —— BufferTag指明一个页在文件系统中的位置，BufferLookupEnt.id则指出了该页在内存buffer中的位置。
```
// SharedBufHash 值的类型
typedef struct
{
	BufferTag	key;			/* Tag of a disk page */
	int			id;				/* Associated buffer ID */  // 也就是BufferBlocks和BufferDescriptors数组的下标。
} BufferLookupEnt;
```

另有两个辅助的共享数据结构:
1. BufferIOLWLockArray  "Buffer IO Locks"  长度固定是NBuffers的数组，每一项都是一个专门用来同步对buffer做io的锁。之所以不把这个锁作为BufferDescriptor的一个字段，是为了要让这些锁本身的内存地址对齐到CPU的cacheline上面。
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
                            

次高4位是usage_count，可以理解成是在最近一小段时间只能对这个页使用的热度，用于clock sweeps算法寻找可以被evict的页。低18位是refcount用来精确记录正在引用(PinBuffer)这个页的进程个数——每个页最多有262144个客户端同时使用。

敏锐的读者可能已经发现，如果postresql只有一个refcount来记录引用buffer页的进程个数，并且对这个数字的加减又是由各个进程主动进行的，那么就会有一个问题: 如果某个backend进程非正常退出，那么backend这个进程就不会unpin(refcount--)它已经pin过的buffer的了，这就导致了buffer泄露！postgresql解决这个问题的做法相当粗暴: 强制杀掉所有"其他"的backend进程，并通知客户端:
```
WARNING:  terminating connection because of crash of another server process
DETAIL:  The postmaster has commanded this server process to roll back the current transaction and exit, because another server process exited abnormally and possibly corrupted shared memory.
HINT:  In a moment you should be able to reconnect to the database and repeat yo
ur command.
```
这就要求开发人员要在客户端程序中处理好这种异常。
这样postgres就有机会把所有buffer的refcount归0。

##### SharedBufHash
这个hash table的key数量比较大(NBuffers + NUM_BUFFER_PARTITIONS 个)，而且需要频繁被多进程访问，pg对此作了一个优化——把它的key空间分成NUM_BUFFER_PARTITIONS(宏定义，默认128)个partition。当需要对某一个key(BufferTag)进行操作时，pg的上锁粒度都是该key所属的partition(计算所属partition的宏是BufTableHashPartition)。


##### 把一个数据文件的页读入buffer
把一个数据文件的页读入buffer(把页的内容从文件原样读入BufferBlocks, 并把 BufferDescriptors和SharedBufHash相应的元素设置正确)的逻辑在bufmgr.c:ReadBuffer_common函数里面实现。"把页读入内存buffer"这件事被分成了两个步骤来做，第一步是找到一个可用的buffer(也就是BufferBlocks数组下标和BufferDescriptors数组下标)，这个步骤由bufmgr.c:BufferAlloc函数实现。第二步是实际进行IO操作把页的内容读入第一步得到buffer中并把BufferDescriptors和SharedBufHash相应的元素设置好。如果要读的这页数据已经在buffer里面了(我们称之为命中buffer)，那么这种情况会模糊两步之间的界限——BufferAlloc通过bool *foundPtr指出是否命中buffer，ReadBuffer_common只有在没有命中buffer的情况下才需要进行IO操作读取该页的内容。
先来分析第一步BufferAlloc。第一步可能有四种结果:
1. 命中buffer——要找的页已经在buffer中。这是最理想的情况。
2. 在1不成立的情况下，可以从freelist(StrategyControl->firstFreeBuffer)中拿出一个空闲的页。这是次理想的情况。
3. 在1和2都不成的情况下，找到一个refcount(BufferDesc.state的低18位)是0，并且usage_count也可以降到0的页。
4. 1,2,3都不成立，直接退出子进程，停止对该客户端继续进行服务。

命中buffer的情况:
```
在bufmgr.c:BufferAlloc函数中:

LWLockAcquire(newPartitionLock, LW_SHARED);    // 以 shared 模式 锁住 SharedBufHash 中BufferTag所在的partition     这个锁本身是在共享结构MainLWLockArray中。 newPartitionLock锁和SharedBufHash并没有固有的内在联系，只是pg中对进行SharedBufHash访问时都要确保正确锁住了partition。整个pg代码中对共享数据结构的使用风格都是如此——访问共享数据结构的代码负责保证用正确的方式上锁，而不是在对共享数据结构操作的函数内部上锁。
buf_id = BufTableLookup(&newTag, newHash);    // 从SharedBufHash找到要读取的页。
if(buf_id) {
    buf = GetBufferDescriptor(buf_id);  
    valid = PinBuffer(buf, strategy);   
    LWLockRelease(newPartitionLock);  // 释放针对SharedBufHash的partition的锁。
    *foundPtr = true;
    if (!valid) {// 这里处理一种比较巧合的并发状况——另一个子进程正好要用这个buffer读取和我们同一个文件系统页(因此我们命中了buffer)，并且读取操作已经失败或者正在进行中(否则BM_VALID位会是1，也就是valid==true)。
        if (StartBufferIO(buf, true))  
        {
            *foundPtr = false;
        }
	}
    return buf;     // 返回找到的BufferDesc *。
}
```

没有命中buffer，但是可以从freelist中拿到空闲buffer的情况:
```
在freelist.c:StrategyGetBuffer函数中:

if (StrategyControl->firstFreeBuffer >= 0)
	{
		while (true)
		{
			
			SpinLockAcquire(&StrategyControl->buffer_strategy_lock);  // 这个锁保护的是StrategyControl

			if (StrategyControl->firstFreeBuffer < 0)     // two-phase lock 模式。
			{
				SpinLockRelease(&StrategyControl->buffer_strategy_lock);
				break;
			}

			buf = GetBufferDescriptor(StrategyControl->firstFreeBuffer);   // 从这里可以看出StrategyControl->firstFreeBuffer是BufferDescriptors数组的下标。
			Assert(buf->freeNext != FREENEXT_NOT_IN_LIST);

			StrategyControl->firstFreeBuffer = buf->freeNext;   //  把freelist里的第一个元素拿掉了。
			buf->freeNext = FREENEXT_NOT_IN_LIST;

			
			SpinLockRelease(&StrategyControl->buffer_strategy_lock);  // 后面不再访问StrategyControl，所以这里就可以释放buffer_strategy_lock了。

			/*
			 * If the buffer is pinned or has a nonzero usage_count, we cannot
			 * use it; discard it and retry.  (This can only happen if VACUUM
			 * put a valid buffer in the freelist and then someone else used
			 * it before we got to it.  It's probably impossible altogether as
			 * of 8.3, but we'd better check anyway.)
			 */
			local_buf_state = LockBufHdr(buf);    //  获取buf的BM_LOCKED锁，并返回BM_LOCKED置成1后的buf->state。 buf的类型是BufferDesc* 。
			if (BUF_STATE_GET_REFCOUNT(local_buf_state) == 0
				&& BUF_STATE_GET_USAGECOUNT(local_buf_state) == 0)   // 再检查一次确保没有被其他进程用着。其实从上面的锁来看，这个if条件应该总是成立的，上面的代码中原本的英文注释佐证了这一点，这个if判断从8.3开始其实就不需要了。
			{
				if (strategy != NULL)
					AddBufferToRing(strategy, buf);
				*buf_state = local_buf_state;
				return buf;    // 注意这里返回的时候buf的BM_LOCKED锁还锁着。
			}
			UnlockBufHdr(buf, local_buf_state);
		}
	}
```

用clock sweep算法从buffer pool中找一个可以给我们用的buffer: 按顺序检查buffer pool中的每一个buffer，如果遇到一个buffer的refcount=0，那么判断它的usagecount，如果usagecount也是0，那么就用这个buffer，否则把它的usagecount减少1，然后对下一个buffer做相同的判断。如果发现buffer pool中没有refcount=0的buffer，则调用elog退出子进程。
```
在freelist.c:StrategyGetBuffer函数中:

trycounter = NBuffers;
for (;;)
{
    buf = GetBufferDescriptor(ClockSweepTick());    //  从逻辑上来说ClockSweepTick相当于: return StrategyControl->nextVictimBuffer  % NBuffers， 但是实际要处理多进程同步问题，并且要更新统计信息。

    /*
     * If the buffer is pinned or has a nonzero usage_count, we cannot use
     * it; decrement the usage_count (unless pinned) and keep scanning.
     */
    local_buf_state = LockBufHdr(buf);

    if (BUF_STATE_GET_REFCOUNT(local_buf_state) == 0)
    {
        if (BUF_STATE_GET_USAGECOUNT(local_buf_state) != 0)
        {
            local_buf_state -= BUF_USAGECOUNT_ONE;

            trycounter = NBuffers;
        }
        else
        {
            // 找到一个可以用的buffer。
            if (strategy != NULL)
                AddBufferToRing(strategy, buf);
            *buf_state = local_buf_state;
            return buf;
        }
    }
    else if (--trycounter == 0)
    {
        // 已经检查过NBuffers个(整个buffer pool里一共有NBuffers个buffer)buffer，没有发现refcount=0的buffer。
        UnlockBufHdr(buf, local_buf_state);
        elog(ERROR, "no unpinned buffers available");  // 退出进程。
    }
    UnlockBufHdr(buf, local_buf_state);
}
```


### 2.3 SeqScan 对表进行顺序扫描
SeqScan是对表中的数据最基本的访问方式，假设我们实现一个只支持SeqScan的数据库——查询规划器query planner只作出使用SeqScan的判断，执行器executor也只会执行SeqScan，那么它也可以被称为"能工作"。在本节暂且不考虑其他的scan方法，也不考虑查询规划器是如何作出选用SeqScan而不选用其他scan方法的决定的，而把注意力放查询执行器是如何执行SeqScan上面。执行器有四个主要的对外(对外是指pg服务器代码的其他部分)接口函数:
```
在execMain.c中
ExecutorStart()   // 在执行一个查询计划之前，做打开文件，分配存储等准备工作。
ExecutorRun()     // 真正执行一个查询计划。
ExecutorFinish()  // 触发after触发器之类的工作。
ExecutorEnd()     // 关闭文件，unpin buffer等工作。
```


查询执行器执行一个查询规划至少要有两个步骤，第一步是
ExecInitNode函数根据查询规划(类型是Plan *) 建立一个 PlanState，第二步是由ExecProcNode函数执行PlanState。

#### ExecInitNode 和 ExecInitSeqScan
```
>	postgres.exe!ExecInitSeqScan(Scan * node, EState * estate, int eflags) Line 154	C
        // 返回一个 SeqScanState *，其中ExecProcNode字段指向ExecSeqScan。
 	postgres.exe!ExecInitNode(Plan * node, EState * estate, int eflags) Line 207	C   
        // 根据nodeTag(node)调用不同的ExecInitXXX，在这个例子中，nodeTag(node) == T_SeqScan，所以调用ExecInitSeqScan。
        // ExecInitXXX 返回的 PlanState* 要为之后运行阶段的 ExecProcNode函数指名了 对应的 ExecProcXXXX，在这个例子中是ExecSeqScan。
 	postgres.exe!InitPlan(QueryDesc * queryDesc, int eflags) Line 1044	C
        // 检查表级权限，根据queryDesc->plannedstmt对queryDesc->estate上的表级资源做初始化。
        // queryDesc->planstate = ExecInitNode返回的PlanState*， 这个例子中是 SeqScanState*;
        // 在运行阶段，这个SeqScanState*会被传给ExecSeqScan和SeqNext。
 	postgres.exe!standard_ExecutorStart(QueryDesc * queryDesc, int eflags) Line 265	C      
        // 创建一个EState *estate，赋值给queryDesc->estate，调用 InitPlan。
 	postgres.exe!ExecutorStart(QueryDesc * queryDesc, int eflags) Line 146	C
        // 只是调用standard_ExecutorStart的入口
 	postgres.exe!PortalStart(PortalData * portal, ParamListInfoData * params, int eflags, SnapshotData * snapshot) Line 525	C
 	postgres.exe!exec_simple_query(const char * query_string) Line 1091	C
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
```
其中的 portal->queryDesc->plannedstmt = portal->stmts->head->data.ptr_value，而portal->stmts 就是规划器(pg_plan_queries)返回的规划结果。

重点看 ExecInitSeqScan
```
SeqScanState *
ExecInitSeqScan(SeqScan *node, EState *estate, int eflags)
{
	SeqScanState *scanstate;

	/*
	 * Once upon a time it was possible to have an outerPlan of a SeqScan, but
	 * not any more.
	 */
	Assert(outerPlan(node) == NULL);
	Assert(innerPlan(node) == NULL);

	/*
	 * create state structure
	 */
	scanstate = makeNode(SeqScanState);
	scanstate->ss.ps.plan = (Plan *) node;
	scanstate->ss.ps.state = estate;
	scanstate->ss.ps.ExecProcNode = ExecSeqScan;

	/*
	 * Miscellaneous initialization
	 *
	 * create expression context for node
	 */
	ExecAssignExprContext(estate, &scanstate->ss.ps);     //  在estate->es_query_cxt 上创建一个ExprContext，并赋值给 scanstate->ss.ps.ps_ExprContext

	/*
	 * Initialize scan relation.
	 *
	 * Get the relation object id from the relid'th entry in the range table,
	 * open that relation and acquire appropriate lock on it.
	 */
	scanstate->ss.ss_currentRelation =
		ExecOpenScanRelation(estate,
							 node->scanrelid,
							 eflags);

	/* and create slot with the appropriate rowtype */
	ExecInitScanTupleSlot(estate, &scanstate->ss,
						  RelationGetDescr(scanstate->ss.ss_currentRelation));   //  创建一个TupleTableSlot，并赋值给 scanstate->ss_ScanTupleSlot。 这个TupleTableSlot用于从表中读数据。

	/*
	 * Initialize result slot, type and projection.
	 */
	ExecInitResultTupleSlotTL(estate, &scanstate->ss.ps);   // 创建一个TupleTableSlot，并赋值给 scanstate->ss.ps.ps_ResultTupleSlot。这个TupleTableSlot用于输出结果。
	ExecAssignScanProjectionInfo(&scanstate->ss);  // 创建一个 ProjectionInfo *, 并赋值给 scanstate->ss.ps.ps_ProjInfo。要注意的是 scanstate->ss.ps.ps_ProjInfo->pi_state.resultslot 和 scanstate->ss.ps.ps_ResultTupleSlot 是相同的地址。

	/*
	 * initialize child expressions
	 */
	scanstate->ss.ps.qual =                     // 这个表示 where 条件。
		ExecInitQual(node->plan.qual, (PlanState *) scanstate);

	return scanstate;
}
```

#### ExecProcNode 和 ExecSeqScan
下面这个stack展示出了main函数到执行器框架代码再到读取buffer的调用关系的全貌。
```
>	postgres.exe!ReadBuffer_common(SMgrRelationData * smgr, char relpersistence, ForkNumber forkNum, unsigned int blockNum, ReadBufferMode mode, BufferAccessStrategyData * strategy, bool * hit) Line 788	C
 	postgres.exe!ReadBufferExtended(RelationData * reln, ForkNumber forkNum, unsigned int blockNum, ReadBufferMode mode, BufferAccessStrategyData * strategy) Line 664	C
 	postgres.exe!heapgetpage(HeapScanDescData * scan, unsigned int page) Line 379	C
        // 把ReadBufferExtended获取到的Buffer 赋值给 scan->rs_cbuf，
        // 如果scan->rs_pageatatime==true，在这个例子里scan->rs_pageatatime确实==true，还要从buffer里找到所有当前snapshot可见的tuple，并把它们序号放到scan->rs_vistuples数组里，同时用scan->rs_ntuples记录这个Buffer中对当前snapshot可见tuple的总个数。
        // 对各条tuple 调用HeapTupleSatisfiesVisibility 判断是否对当前snapshot 可见(visible)。MVCC 的判断就是在这里做的。
 	postgres.exe!heapgettup_pagemode(HeapScanDescData * scan, ScanDirection dir, int nkeys, ScanKeyData * key) Line 838	C
        //  TestForOldSnapshot 检查当前的snapshot 对整个页而言是不是太老。
        //  
 	postgres.exe!heap_getnext(HeapScanDescData * scan, ScanDirection direction) Line 1842	C
 	postgres.exe!SeqNext(SeqScanState * node) Line 80	C
 	postgres.exe!ExecScanFetch(ScanState * node, TupleTableSlot *(*)(ScanState *) accessMtd, bool(*)(ScanState *, TupleTableSlot *) recheckMtd) Line 96	C     // 
 	postgres.exe!ExecScan(ScanState * node, TupleTableSlot *(*)(ScanState *) accessMtd, bool(*)(ScanState *, TupleTableSlot *) recheckMtd) Line 162	C    // ExecScanFetch，判断是否符合where条件(qual)，对字段做project，project结果存到 node->ps.ps_ProjInfo->pi_state.resultslot上，并返回node->ps.ps_ProjInfo->pi_state.resultslot。如果不需要做project，也没有where条件，则直接返回ExecScanFetch的结果。
 	postgres.exe!ExecSeqScan(PlanState * pstate) Line 132	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecutePlan(EState * estate, PlanState * planstate, bool use_parallel_mode, CmdType operation, bool sendTuples, unsigned __int64 numberTuples, ScanDirection direction, _DestReceiver * dest, bool execute_once) Line 1721	C
 	postgres.exe!standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 376	C
 	postgres.exe!ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 306	C
 	postgres.exe!PortalRunSelect(PortalData * portal, bool forward, long count, _DestReceiver * dest) Line 934	C
 	postgres.exe!PortalRun(PortalData * portal, long count, bool isTopLevel, bool run_once, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 773	C
 	postgres.exe!exec_simple_query(const char * query_string) Line 1130	C
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```
重点看 ExecSeqScan，它是执行器框架代码和SeqScan之间的枢纽:
```
static TupleTableSlot *   // 上一章已经看过，ExecSeqScan返回的TupleTableSlot *表示要返回给客户端的一个row。
ExecSeqScan(PlanState *pstate)
{
	SeqScanState *node = castNode(SeqScanState, pstate);

	return ExecScan(&node->ss,
					(ExecScanAccessMtd) SeqNext,
					(ExecScanRecheckMtd) SeqRecheck);
}
```
ExecSeqScan的实现很简单：以SeqNext和SeqRecheck调用ExecScan。目前我们只要知道ExecScan和ExecScanFetch都是执行其框架中所有scan方法的共同部分，所以直接看SeqNext。

```
static TupleTableSlot *
SeqNext(SeqScanState *node)
{
	HeapTuple	tuple;
	HeapScanDesc scandesc;
	EState	   *estate;
	ScanDirection direction;
	TupleTableSlot *slot;

	/*
	 * get information from the estate and scan state
	 */
	scandesc = node->ss.ss_currentScanDesc;
	estate = node->ss.ps.state;
	direction = estate->es_direction;
	slot = node->ss.ss_ScanTupleSlot;

	if (scandesc == NULL)
	{
		/*
		 * We reach here if the scan is not parallel, or if we're serially
		 * executing a scan that was planned to be parallel.
		 */
		scandesc = heap_beginscan(node->ss.ss_currentRelation,   // 创建一个HeapScanDesc。
								  estate->es_snapshot,
								  0, NULL);             //  注意这里固定没有任何ScanKey，因为where条件的检查是在ExecScan做的。实际上10.0的代码里面所有对heap_beginscan的调用都没有指定任何ScanKey。
		node->ss.ss_currentScanDesc = scandesc;
	}

	/*
	 * get the next tuple from the table
	 */
	tuple = heap_getnext(scandesc, direction);    // 从这里可以看出，SeqScan只是简单地调用heapscan。在pg中，把存放表的数据的数据结构叫做heap——特指数据文件而不是索引，heapscan就是指对数据文件(当然实际是内存中的buffer)直接扫描。

	/*
	 * save the tuple and the buffer returned to us by the access methods in
	 * our scan tuple slot and return the slot.  Note: we pass 'false' because
	 * tuples returned by heap_getnext() are pointers onto disk pages and were
	 * not created with palloc() and so should not be pfree()'d.  Note also
	 * that ExecStoreTuple will increment the refcount of the buffer; the
	 * refcount will not be dropped until the tuple table slot is cleared.
	 */
     // 上面的英文注释中只是说ExecStoreTuple要增加对buffer的refcount，这有可能让人误以为是对BufferDesc.state字段增加计数，实际上ExecStoreTuple是调用IncrBufferRefCount增加私有(本进程私有，而非进程间共享)引用计数。
	if (tuple)    // tuple的类型是HeapTuple，但是需要返回一个TupleTableSlot *，所以调用ExecStoreTuple把tuple填到slot里面。
		ExecStoreTuple(tuple,	/* tuple to store */
					   slot,	/* slot to store in */
					   scandesc->rs_cbuf,	/* buffer associated with this
											 * tuple */
					   false);	/* don't pfree this pointer */
	else
		ExecClearTuple(slot);

	return slot;
}
```
下面分析对heap数据的操作，需要首先阅读官方文档66.6节Database page layout 作为基础。
其中HeapTuple的定义如下:
```
typedef struct HeapTupleData
{
	uint32		t_len;			// *t_data的长度， 包含了23字节的HeapTupleHeaderData的长度加上null bitmap的长度再加上t_hoff指向的实际数据的长度。
	ItemPointerData t_self;		// block id 和 ItemIdData 数组的下标。
	Oid			t_tableOid;		// 这个tuple所属的表的oid。在pg里面，每个表都有一个oid作为标识。
#define FIELDNO_HEAPTUPLEDATA_DATA 3
	HeapTupleHeader t_data;		// 直接指向这个page对应的buffer中的tuple(对应66.6中Items数组的元素)。t_data->t_hoff偏移量是针对 t_data本身而言，也就是说这个tuple的用户数据的实际内存地址是 t_data + t_data->t_hoff。
} HeapTupleData;
```
官方文档66.6节只是说ItemId一共4字节，包含实际item的偏移量和长度，但是没有给出具体二进制布局。 从代码里找到ItemId的二进制布局如下, 还有一个lp_flags字段。
```
typedef struct ItemIdData
{
	unsigned	lp_off:15,		// tuple(HeapTupleData)相对page起始的偏移量。
				lp_flags:2,		
				lp_len:15;		// tuple的总长度。
} ItemIdData;

typedef ItemIdData *ItemId;
// 下面4个常量是lp_flags肯能的取值。
#define LP_UNUSED		0		// 未使用。
#define LP_NORMAL		1		// 正常使用状态。lp_len一定大于0。
#define LP_REDIRECT		2		// HOT(heap only tuple) redirect lp_len应该等于0。
#define LP_DEAD			3		// 死的，可能占存储空间也可能不占。是指这个ItemIdData所指的item已经不对任何事务可见了，item可能已经被recyle了，但是这个ItemIdData本身还被索引引用，尚未被recycle。
```

下面展示了heapgettup_pagemode从page buffer构建一个HeapTupleData的关键代码，page是页号，dp是改页对应的共享buffer中的地址:
```
lineoff = scan->rs_vistuples[lineindex];  // lineoff 是 页头中 ItemIdData 数组的下标。lineoff以1为起始。
lpp = PageGetItemId(dp, lineoff); // lpp 的类型是ItemId，也就是ItemIdData *。
Assert(ItemIdIsNormal(lpp));

tuple->t_data = (HeapTupleHeader) PageGetItem((Page) dp, lpp);  //  t_data就在 dp + lpp->lp_off。tuple->t_data指向这一个row在共享buffer中的地址，tuple->t_data + tuple->t_data->t_hoff 则是这个row的用户数据所在的地址。
tuple->t_len = ItemIdGetLength(lpp);                    // 把 lpp->lp_len赋值给 tuple->t_len。
ItemPointerSet(&(tuple->t_self), page, lineoff);        // 把lineoff 赋值给tuple->t_self.ip_posid; 把page赋值给tuple->t_self.ip_blkid
```
tuple

还有ExecutorFinish()和ExecutorEnd()，不过本章的目的是帮读者建立一个比较清晰的大局观，所以暂且忽略这些细节。


[下一章](chap3.md)
