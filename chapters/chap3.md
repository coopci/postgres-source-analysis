## ������  insert, delete, update��ִ�� ##


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
            // �쿴 (ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree) �����ݡ�
            // ��һ������(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr�� ���е�funcid�ֶε�ֵ��1574
            // SELECT  oid, * FROM pg_proc where oid = 1574 ==> nextval
            
            // �ڶ������� (Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr�� ���е�consttype�ֶε�ֵ��1043
            // select oid, * from pg_type where oid = 1043; ==> varchar
            // ֪����varchar֮�������watch���� (char*)((Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr)->constvalue + 4 ����ַ���������ʲô�����ֹ�Ȼ��"foo"
            // ((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist �������Ҫ��������ݡ�
            
            
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```



Thus, a tuple is the latest version of its row iff XMAX is invalid or

### �ܽ�
ǰ����һ������������һ���������������չʾ��pg�Կͻ������ӣ�����buffer�������ڴ棬�����ļ�buffer�Ļ����÷���heap�Ļ�����֯�ṹ��Ҳ����pg��Ϊһ��C/S�ṹ�����ݴ�ȡ�������Ļ���ʵ�֡��ӵ����¿�ʼ������pg��Ϊһ����ϵ�����ݿ����ϵͳ�����������
