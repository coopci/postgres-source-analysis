## 第一章  粗筋 ##

1.1  走马观花一番 select * from table1;

本节向读者展示pg服务器在处理一个简单的 select * 查询的过程所包含的主要步骤，让读者在源代码层次对大框架产生初步印象。功能模块角度的大框架可以从官方文档的第五十章找到。

手工操作: 用客户端程序 psql  建立一个新连接: 
```
psql -U postgres -p 6543
```

这里 pg主进程 accept一个新的tcp连接，并fork一个子进程处理这个连接的后续请求。主进程回到 listen, accept, fork 循环。

```
postgres.exe!BackendStartup(Port * port) Line 4016	C        ==> pid = backend_forkexec(port); 我们需要记住这个pid， 后面要把debugger attach到这进程上。
postgres.exe!ServerLoop() Line 1712	C
postgres.exe!PostmasterMain(int argc, char * * argv) Line 1379	C
postgres.exe!main(int argc, char * * argv) Line 229	C
```

把debuger  attach 到 上面记下 pid 上，然后从psql 里面 发送一个查询命令，例如 select * from table1;
```
>	postgres.exe!secure_raw_read(Port * port, void * ptr, unsigned __int64 len) Line 233	C     这里是从tcp连接接受数据的地方。
 	postgres.exe!secure_read(Port * port, void * ptr, unsigned __int64 len) Line 158	C
 	postgres.exe!pq_recvbuf() Line 963	C
 	postgres.exe!pq_getbyte() Line 1006	C           注意这里。
 	postgres.exe!SocketBackend(StringInfoData * inBuf) Line 341	C
 	postgres.exe!ReadCommand(StringInfoData * inBuf) Line 514	C
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4095	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
```

支线剧情: 从 pq_getbyte 可以发现 pqcomm.c 里面有这么个全局变量 static char PqRecvBuffer[PQ_RECV_BUFFER_SIZE]，它 是接收客户端原始数据的buffer。

回到主线:       qtype是用 pq_getbyte 从 接收buffer里获取的第一个字节。
```
>	postgres.exe!SocketBackend(StringInfoData * inBuf) Line 375	C     根据 qtype 来决定之后的路线。如果 qtype==81(也就是'Q') 表示buffer里当前的命令是一个  simple query。  具体的协议规范可以在postgresl-10 的官方文档的 52.7节 Message Formats 找到。
 	postgres.exe!ReadCommand(StringInfoData * inBuf) Line 514	C
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4095	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
 	[External Code]	
```
一步一步走到下一个断点:
```    
>	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4147	C   在这里看到query_string里面就是刚刚从psql发过来的命令。
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
```

```
>	postgres.exe!exec_simple_query(const char * query_string) Line 944	C                   这里要调用语法分析器(parser)从原始查询(query_string)生成一个语法树(parsetree_list)
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
```

这里的pg_parse_query 返回 parsetree_list 的类型是List*:
List 的定义在 pg_list.h， 如下:
```
typedef struct List
{
	NodeTag		type;			/* T_List, T_IntList, or T_OidList */
	int			length;
	ListCell   *head;
	ListCell   *tail;
} List;
```
List 的元素 ListCell类型定义是这样:
```
struct ListCell
{
	union
	{
		void	   *ptr_value;
		int			int_value;
		Oid			oid_value;
	}			data;
	ListCell   *next;
};
```
发现实际的内容是 void *ptr_value。接下来就需要确定ptr_value的数据类型。
在 postgres.c:976 可以发现: 

```
RawStmt *parsetree = lfirst_node(RawStmt, parsetree_item);
parsetree	0x00000000006366e8 {type=T_RawStmt (226) stmt=0x00000000006365d8 {type=T_SelectStmt (232) } stmt_location=...}	RawStmt *
```

于是可以画出如下的结构:
```
List* parsetree_list
           |
  +--------+
  |         
  V
-----------------------------
|NodeTag()|length| head|tail|   
-----------------------------
                     |
    +----------------+                 
    |
    |    -----------------------  
    +--->| data/ptr_value| next|   ListCell
         -----------------------
                  |         |
                  |         +--> null
                  |  ------------------
                  +->| NodeTag()| stmt|   RawStmt  对应代码中的 parsetree 变量。
                     ------------------
                                    |     -------------------------
                                    +---->|NodeTag()|   |   | ....|  SelectStmt
                                          -------------------------
```
最后这个 SelectStmt 就是我们的查询语句 select * from table1 对应的语法树。用 (SelectStmt*)parsetree->stmt 来Watch 一下就可以大致知道里面有哪些内容。
两句题外话：这种用nodetag来标记实际类型， 并以此为依据进行指针cast的做法是C语言项目里一种常用的面向对象编程方法。
其实面向对象语言不外乎是在语法层面通过编译器或者VM替程序员向内存块里嵌入了这种表示类型的tag。

接下来这个位置上两个目前比较重要的变量是 querytree_list 和 plantree_list。 
querytree_list 是 应用了rewrite 规则后得到结果。例如把针对视图的查询分解成针对相应表的查询就是一种rewrite规则。
plantree_list 是 以querytree_list做为输入 进行查询规划后得到的结果。所谓的查询优化就是在这个步骤完成的。
我们可以直接看到这两个list都是分别只有一个元素(head和tail相同)。
```
>	postgres.exe!exec_simple_query(const char * query_string) Line 1054	C
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
 	[External Code]	
```    
有了之前看parsetree的经验，我们可以用这两个watch:
```
(Node*)(plantree_list->head->data.ptr_value)
(Node*)(querytree_list->head->data.ptr_value)
```
来看看这两个元素具体是什么类型。可以看到结果分别是T_Query和T_PlannedStmt:
```
(Node*)(plantree_list->head->data.ptr_value)	0x0000000000627820 {type=T_PlannedStmt (228) }	Node *
(Node*)(querytree_list->head->data.ptr_value)	0x00000000006271d8 {type=T_Query (227) }	Node *
```
知道了具体类型后，就可以用这两个watch看看它们里面的具体内容:
```
(Query*)(querytree_list->head->data.ptr_value)	0x00000000006271d8 {type=T_Query (227) commandType=CMD_SELECT (1) querySource=QSRC_ORIGINAL (0) ...}	Query *
(PlannedStmt*)(plantree_list->head->data.ptr_value)	0x0000000000627820 {type=T_PlannedStmt (228) commandType=CMD_SELECT (1) queryId=0 ...}	PlannedStmt *
```



下面这个stack是pg在做SeqScan时发现用到的那个页目前还不在共享内存中，从文件读取该页的代码。共享内存和页buffer的组织方式会在第二章做比较详细的分析。
```
>	postgres.exe!ReadBuffer_common(SMgrRelationData * smgr, char relpersistence, ForkNumber forkNum, unsigned int blockNum, ReadBufferMode mode, BufferAccessStrategyData * strategy, bool * hit) Line 788	C
 	postgres.exe!ReadBufferExtended(RelationData * reln, ForkNumber forkNum, unsigned int blockNum, ReadBufferMode mode, BufferAccessStrategyData * strategy) Line 664	C
 	postgres.exe!heapgetpage(HeapScanDescData * scan, unsigned int page) Line 379	C
 	postgres.exe!heapgettup_pagemode(HeapScanDescData * scan, ScanDirection dir, int nkeys, ScanKeyData * key) Line 838	C
 	postgres.exe!heap_getnext(HeapScanDescData * scan, ScanDirection direction) Line 1842	C
 	postgres.exe!SeqNext(SeqScanState * node) Line 80	C
 	postgres.exe!ExecScanFetch(ScanState * node, TupleTableSlot *(*)(ScanState *) accessMtd, bool(*)(ScanState *, TupleTableSlot *) recheckMtd) Line 96	C
 	postgres.exe!ExecScan(ScanState * node, TupleTableSlot *(*)(ScanState *) accessMtd, bool(*)(ScanState *, TupleTableSlot *) recheckMtd) Line 162	C
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
    
下面这个stack顶部的printtup是pg找到一个符合查询条件的row(用TupleTableSlot *表示一个row)，向客户端输出(存入发送buffer或者调用tcp发送操作)这个row。
```
>	postgres.exe!printtup(TupleTableSlot * slot, _DestReceiver * self) Line 376	C
 	postgres.exe!ExecutePlan(EState * estate, PlanState * planstate, bool use_parallel_mode, CmdType operation, bool sendTuples, unsigned __int64 numberTuples, ScanDirection direction, _DestReceiver * dest, bool execute_once) Line 1756	C
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

针对我们这个简单的select * 示例，下面的ReadyForQuery是pg服务器在告诉客户端“之前的请求都处理完了，现在可以发送新的请求了”。
```
>	postgres.exe!ReadyForQuery(CommandDest dest) Line 259	C   // 向客户端发送ReadyForQuery消息。
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4081	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C
```
在这之间pg服务器还要向客户端发送一个CommandComplete消息，找到发送CommandComplete消息的代码以及stack的位置留给读者作为练习。如果目前没有思路，可以使用1.2节介绍的方法。
至此，我们已经看过了一个最简单的查询语句的整体执行流程，之后的一部分章节是对这个粗略描述的细化补充。

1.2 C/S通信部分的详细分析
之所以把这个部分的详细分析放到"粗筋"章，有几个原因:
1. 这个部分相对简单，可以在比较少的篇幅解释清楚。
2. 这个部分的行为可以从外部观察到(使用wireshark或者tcpdump之类的工具)，或者说pg服务器的所有其他内部逻辑的存在都是为了实现这个通信。 尽快交待清楚可以方便读者使用逆推的方法寻找感兴趣的部分代码。例如 如果想要知道pg是如何计算出一条应该返回给客户端的数据的，那么可以首先在发送DataRow消息(消息类型是'D') 的地方设置断点，然后根据这断点的stack就可以更方便地探索感兴趣的部分。

pg的通信协议以消息(postgresl-10 的官方文档的 52.7节)为最小的语义单位，每个消息的第一个字节都用来表示本消息的类型，第2到第5个字节表示本消息的长度，从第6个字节开始是消息的内容。pg服务器端的代码用StringInfo结构(stringinfo.h)在内存中组装要发送的消息，然后通过pq_endmessage_reuse函数把组装好的StringInfo结构发送给接收方。需要注意的是，消息类型会被存到StringInfo的cursor字段，长度存到len字段，data字段指向的内存是是消息的第6个字节。

libpq/pgcomm.c 中的 static char PqRecvBuffer[PQ_RECV_BUFFER_SIZE] 用来接收来自客户端的消息序列，static char *PqSendBuffer 用来作为发给客户端的消息的buffer。

这里分析一种PqSendBuffer的典型用法:
```
access/common/printtup.c:printtup -> // 把当前要输出的这个TupleTableSlot(也就是作为查询结果的一个row)装到一个StringInfo里面，然后 以这个StringInfo 为参数调用pq_endmessage_reuse。printtup函数要作为DestReceiver的receiveSlot成员被调用。
    libpq/pgformat.c:pq_endmessage_reuse -> pq_putmessage(buf->cursor, buf->data, buf->len);
        libpq.h:pq_putmessage -> // 这是个宏，宏定义是PqCommMethods->putmessage函数指针，这个函数指针指向libpq/pgcomm.c:socket_putmessage
            libpq/pgcomm.c:socket_putmessage ->  // 把StringInfo转化为DataRow的形式输出
                libpq/pgcomm.c:internal_putbytes ->  // 把上层要输出的字节流 memcpy 到 PqSendBuffer 中，如果PqSendBuffer满了，则调用internal_flush。
                    libpq/pgcomm.c:internal_flush  // 把字节流写入socket。
```
在printtup里面，有一个叫做pq_beginmessage_reuse的函数和pq_endmessage_reuse配对出现，它的作用是把StringInfo 重置成类似于初始化的状态(len字段设置为0)，并指明了这个StringInfo 在本次reuse的过程中是准备用来构造什么类型的消息(设置cursor字段)。 这个代码结构给我们提供了一个非常方便的线索 —— socket_putmessage 作为构造输出消息的起始点，同时也就是pg各种内部逻辑暂时告一段落的位置。所以如果从分析代码结构的角度来考虑，socket_putmessage 是设置断点并观察调用栈的好位置。

[下一章](chap2.md)
