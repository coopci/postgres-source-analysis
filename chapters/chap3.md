## 第三章  insert, delete, update的执行 ##


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
        // 在当前MemoryContext里分配一个HeapTuple，并根据tupleDescriptor对各字段的描述把values给定的各字段的值，拷贝到分配好的HeapTuple中。这个HeapTuple不在共享buffer中，之后还要把它整个拷贝的到共享buffer中。
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
            // 第一个参数(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr， 其中的funcid字段的值是1574
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
