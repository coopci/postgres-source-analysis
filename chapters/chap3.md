## 第三章  insert, delete, update的执行 ##


### insert
```
insert into table1(col1) values ('foo');
```

```
>	postgres.exe!heap_form_tuple(tupleDesc * tupleDescriptor, unsigned __int64 * values, bool * isnull) Line 1082	C
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
            // 察看 (ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree) 的内容。
            // 第一个参数(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr， 其中的funcid字段的值是1574
            // SELECT  oid, * FROM pg_proc where oid = 1574 ==> nextval
            
            // 第二个参数 (Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr， 其中的consttype字段的值是1043
            // select oid, * from pg_type where oid = 1043; ==> varchar
            // 知道是varchar之后，用这个watch看看 (char*)((Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr)->constvalue + 4 这个字符串到底是什么，发现果然是"foo"
            // ((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist 用来存放要插入的数据。
            
            
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```



Thus, a tuple is the latest version of its row iff XMAX is invalid or

### 总结
前三章一起给读者描绘了一个轮廓，这个轮廓展示了pg对客户端连接，网络buffer，共享内存，数据文件buffer的基本用法和heap的基本组织结构。也就是pg作为一个C/S结构的数据存取服务器的基本实现。从第四章开始，将对pg作为一个关系型数据库管理系统做深入分析。
