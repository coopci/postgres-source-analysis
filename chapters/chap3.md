## 第三章 执行器 和 insert, delete, update的执行 ##

### 执行器简介
在postgres中，执行器(backend/executor)的工作是根据规划器(pg_plan_queries)返回的查询计划(Plan* PlannedStmt.planTree)执行查询。
查询计划是个树形结构，每个节点被调用时都要返回下一个元组(tuple)，如果没有元组可返回，则返回NULL。如果一个节点还有子节点，那么它要先获取子节点的元组，然后这些自节点返回的元组作为数据源进行自己的运算后再返回-- 例如一个排序节点(struct Sort)要从它的子节点获取所有的元组然后排序，之后才能在每次被调用时从排好序的元组中返回下一个。 这样执行器执行树根的元素，其下各层子节点就会自动递归执行自己负责的部分，最终保证根节点的顺利执行。



执行器本身对postgres的其他组建提供了四个接口: ExecutorStart, ExecutorRun, ExecutorFinish 和 ExecutorEnd。ProcessQuery 函数负责构建一个QueryDesc，并以QueryDesc为参数调用执行器的四个接口函数，上面提到的查询计划也是作为QueryDesc的字段传给执行器的。

QueryDesc =plannedstmt=> (PlannedStmt*) =planTree=> (Plan *)


查询计划对于执行器来说是个只读的数据结构。但是随着执行进度推进，势必会使当前这次"执行"的内部状态发生改变——例如执行扫描表的计划时要记录当前扫描到了哪条记录。postgres的做法是为查询计划树建立一个"平行"的查询状态树(PlanState*) -- 每一个查询计划树中的节点都有一个对应的查询状态树的结点，而且查询状态树的结点记录(plan字段)自己对应哪个查询计划树的结点




ExecutorStart
    InitPlan(QueryDesc)
        ExecInitNode 根据Plan*初始化对应的PlanState*
ExecutorRun 对根节点调用ExecutePlan
   
### insert
```
insert into table1(col1) values ('foo');
```

初始化目标表:
```
>	postgres.exe!InitPlan(QueryDesc * queryDesc, int eflags) Line 849	C
        // 用 getrelid 获得目标表的oid。 从getrelid的实现可以看出: resultRelations是以1为起始。
        // 用 heap_open 以RowExclusiveLock模式打开Relation。并放到estate->es_result_relations里面。
 	postgres.exe!standard_ExecutorStart(QueryDesc * queryDesc, int eflags) Line 265	C
 	postgres.exe!ExecutorStart(QueryDesc * queryDesc, int eflags) Line 146	C
 	postgres.exe!ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfoData * params, QueryEnvironment * queryEnv, _DestReceiver * dest, char * completionTag) Line 161	C
 	postgres.exe!PortalRunMulti(PortalData * portal, bool isTopLevel, bool setHoldSnapshot, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 1291	C
 	postgres.exe!PortalRun(PortalData * portal, long count, bool isTopLevel, bool run_once, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 803	C
 	postgres.exe!exec_simple_query(const char * query_string) Line 1130	C
        // 这里我们要弄清楚要插入的数据和目标表是如何表示的，首先是要插入的数据：
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable 里面记录着这个查询所可能用到的所有的表，在我们这个例子中只用到了一个表:
        ((RangeTblEntry*)((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable->head->data.ptr_value)->relid 和 SELECT oid, * FROM pg_catalog.pg_class where relname='table1'; 得到oid正好吻合。
        
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations 里面记录着insert的目标表在rttable中的索引，用这个watch可以看到它的值是1:
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations->head->data.int_value，
    
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```

execute
```
>	postgres.exe!heap_fill_tuple(tupleDesc * tupleDesc, unsigned __int64 * values, bool * isnull, char * data, unsigned __int64 data_size, unsigned short * infomask, unsigned char * bit) Line 345	C
        // 向下面三个地址中填入正确的数据，三个地址对应的数据 参考HeapTupleHeaderData结构和官方文档的table 66.4。
        // char *data  td + hoff  对应真正的用户数据。
        // uint16 *infomask,   &td->t_infomask
        // bits8 *bit  td->t_bits   对应null bitmap
 	postgres.exe!heap_form_tuple(tupleDesc * tupleDescriptor, unsigned __int64 * values, bool * isnull) Line 1157	C
        // 在当前MemoryContext里分配一个HeapTuple，并根据tupleDescriptor对各字段的描述把values给定的各字段的值，拷贝到分配好的HeapTuple中。这个HeapTuple不在共享buffer中，之后还要把它整个拷贝到共享buffer中。
        //
 	postgres.exe!ExecCopySlotTuple(TupleTableSlot * slot) Line 603	C
 	postgres.exe!ExecMaterializeSlot(TupleTableSlot * slot) Line 806	C
 	postgres.exe!ExecInsert(ModifyTableState * mtstate, TupleTableSlot * slot, TupleTableSlot * planSlot, EState * estate, bool canSetTag) Line 283	C
 	postgres.exe!ExecModifyTable(PlanState * pstate) Line 2161	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecutePlan(EState * estate, PlanState * planstate, bool use_parallel_mode, CmdType operation, bool sendTuples, unsigned __int64 numberTuples, ScanDirection direction, _DestReceiver * dest, bool execute_once) Line 1721	C
 	postgres.exe!standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 376	C
 	postgres.exe!ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 306	C
 	postgres.exe!ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfoData * params, QueryEnvironment * queryEnv, _DestReceiver * dest, char * completionTag) Line 166	C
 	postgres.exe!PortalRunMulti(PortalData * portal, bool isTopLevel, bool setHoldSnapshot, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 1291	C
 	postgres.exe!PortalRun(PortalData * portal, long count, bool isTopLevel, bool run_once, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 803	C
 	postgres.exe!exec_simple_query(const char * query_string) Line 1130	C
            // 这里我们要弄清楚要插入的数据是如何表示的：
            // 察看 (ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree) 的内容。
            // 第一个参数(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr， 其中的 funcid 字段的值是1574
            // SELECT  oid, * FROM pg_proc where oid = 1574 ==> nextval
            
            // 第二个参数 (Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr， 其中的consttype字段的值是1043
            // select oid, * from pg_type where oid = 1043; ==> varchar
            // 知道是varchar之后，用这个watch看看 (char*)((Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr)->constvalue + 4 这个字符串到底是什么，发现果然是"foo"。
            
            // ((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist 用来存放要插入到表中的数据。
            
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```
postgresql以NodeTag为基础，构建了一套"动态"类型机制。如果严格用C语言风格的类型转换进行表示，就会像上面的watch表达式一样难以看懂，所以笔者发明了以下这种更易于人类阅读的方式来描述运行时的数据结构。这种表示形式没有严格区分指针和结构。
用 "类型1 =字段a=> 类型2{值}" 来表示 类型1的字段a的数据类型是类型2，{}里面则是类型2的值。
如果 字段a的C语言类型是List*，那么用  "=字段a[n]=>" 的形式来表示 List*的第n个元素 是箭头右边的类型。
"类型1
     \=字段b=> 类型2" 的形式 同样表示 类型1的字段b的数据类型是类型2。

以上面的insert为例，我们可以把数据结构这样更直观地描述出来:
                                         
Portal =stmts[0]=> (PlannedStmt*) =planTree=> (ModifyTable*) =plans[0]=> (Result*) =plan.targetlist[0]=> (TargetEntry*) =expr=> (FuncExpr*) =funcid=> Oid{1574}
                                                                                  \=plan.targetlist[1]=> (TargetEntry*) =expr=> (Const*) =consttype=> Oid{1043}
                                                                                                                                         \=constvalue=> Datum{"foo"}

ExecInitModifyTable 构造 ModifyTableState -- 对 ModifyTable的plans中的Plan执行ExecInitNode，并把得到的PlanState结果存到ModifyTableState的mt_plans中。

ModifyTableState =ps.plan=> (ModifyTable*)
                 \=mt_plans[0]=> (ResultState*)
                 \=mt_nplans=> int (list_length(ModifyTable->plans))， 数组mt_plans中有几个元素，这个例子中是1，也就是说mt_plans中只有一个元素--ResultState
                 \=ps.ExecProcNode=> ExecModifyTable  

下面来看看 ExecModifyTable 是如何递归调用 ExecResult
```
>	postgres.exe!ExecResult(PlanState * pstate) Line 77	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecModifyTable(PlanState * pstate) Line 2027	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	这里忽略和之前相同的stack。
```

    
```
>	postgres.exe!RelationPutHeapTuple(RelationData * relation, int buffer, HeapTupleData * tuple, bool token) Line 53	C
 	postgres.exe!heap_insert(RelationData * relation, HeapTupleData * tup, unsigned int cid, int options, BulkInsertStateData * bistate) Line 2490	C
        //调用 heap_prepare_insert 处理TOAST。
        // 
 	postgres.exe!ExecInsert(ModifyTableState * mtstate, TupleTableSlot * slot, TupleTableSlot * planSlot, EState * estate, bool canSetTag) Line 529	C
        // 调用 heap_insert 把 ExecMaterializeSlot 得到的tuple写到page里(共享buffer)。
 	
    这里忽略和之前相同的stack。
```
    
Thus, a tuple is the latest version of its row iff XMAX is invalid or

### 总结
前三章一起给读者描绘了一个轮廓，这个轮廓展示了pg对客户端连接，网络buffer，共享内存，数据文件buffer的基本用法和heap的基本组织结构。也就是pg作为一个C/S结构的数据存取服务器的基本实现。从第四章开始，将对pg作为一个关系型数据库管理系统做深入分析。
