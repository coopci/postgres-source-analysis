## ������ ִ���� �� insert, delete, update��ִ�� ##

### ִ�������
��postgres�У�ִ����(backend/executor)�Ĺ����Ǹ��ݹ滮��(pg_plan_queries)���صĲ�ѯ�ƻ�(Plan* PlannedStmt.planTree)ִ�в�ѯ��
��ѯ�ƻ��Ǹ����νṹ��ÿ���ڵ㱻����ʱ��Ҫ������һ��Ԫ��(tuple)�����û��Ԫ��ɷ��أ��򷵻�NULL�����һ���ڵ㻹���ӽڵ㣬��ô��Ҫ�Ȼ�ȡ�ӽڵ��Ԫ�飬Ȼ����Щ�Խڵ㷵�ص�Ԫ����Ϊ����Դ�����Լ���������ٷ���-- ����һ������ڵ�(struct Sort)Ҫ�������ӽڵ��ȡ���е�Ԫ��Ȼ������֮�������ÿ�α�����ʱ���ź����Ԫ���з�����һ���� ����ִ����ִ��������Ԫ�أ����¸����ӽڵ�ͻ��Զ��ݹ�ִ���Լ�����Ĳ��֣����ձ�֤���ڵ��˳��ִ�С�



ִ���������postgres�������齨�ṩ���ĸ��ӿ�: ExecutorStart, ExecutorRun, ExecutorFinish �� ExecutorEnd��ProcessQuery �������𹹽�һ��QueryDesc������QueryDescΪ��������ִ�������ĸ��ӿں����������ᵽ�Ĳ�ѯ�ƻ�Ҳ����ΪQueryDesc���ֶδ���ִ�����ġ�

QueryDesc =plannedstmt=> (PlannedStmt*) =planTree=> (Plan *)


��ѯ�ƻ�����ִ������˵�Ǹ�ֻ�������ݽṹ����������ִ�н����ƽ����Ʊػ�ʹ��ǰ���"ִ��"���ڲ�״̬�����ı䡪������ִ��ɨ���ļƻ�ʱҪ��¼��ǰɨ�赽��������¼��postgres��������Ϊ��ѯ�ƻ�������һ��"ƽ��"�Ĳ�ѯ״̬��(PlanState*) -- ÿһ����ѯ�ƻ����еĽڵ㶼��һ����Ӧ�Ĳ�ѯ״̬���Ľ�㣬���Ҳ�ѯ״̬���Ľ���¼(plan�ֶ�)�Լ���Ӧ�ĸ���ѯ�ƻ����Ľ��




ExecutorStart
    InitPlan(QueryDesc)
        ExecInitNode ����Plan*��ʼ����Ӧ��PlanState*
### insert
```
insert into table1(col1) values ('foo');
```

��ʼ��Ŀ���:
```
>	postgres.exe!InitPlan(QueryDesc * queryDesc, int eflags) Line 849	C
        // �� getrelid ���Ŀ����oid�� ��getrelid��ʵ�ֿ��Կ���: resultRelations����1Ϊ��ʼ��
        // �� heap_open ��RowExclusiveLockģʽ��Relation�����ŵ�estate->es_result_relations���档
 	postgres.exe!standard_ExecutorStart(QueryDesc * queryDesc, int eflags) Line 265	C
 	postgres.exe!ExecutorStart(QueryDesc * queryDesc, int eflags) Line 146	C
 	postgres.exe!ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfoData * params, QueryEnvironment * queryEnv, _DestReceiver * dest, char * completionTag) Line 161	C
 	postgres.exe!PortalRunMulti(PortalData * portal, bool isTopLevel, bool setHoldSnapshot, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 1291	C
 	postgres.exe!PortalRun(PortalData * portal, long count, bool isTopLevel, bool run_once, _DestReceiver * dest, _DestReceiver * altdest, char * completionTag) Line 803	C
 	postgres.exe!exec_simple_query(const char * query_string) Line 1130	C
        // ��������ҪŪ���Ҫ��������ݺ�Ŀ�������α�ʾ�ģ�������Ҫ��������ݣ�
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable �����¼�������ѯ�������õ������еı����������������ֻ�õ���һ����:
        ((RangeTblEntry*)((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable->head->data.ptr_value)->relid �� SELECT oid, * FROM pg_catalog.pg_class where relname='table1'; �õ�oid�����Ǻϡ�
        
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations �����¼��insert��Ŀ�����rttable�е������������watch���Կ�������ֵ��1:
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations->head->data.int_value��
    
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```

execute
```
>	postgres.exe!heap_fill_tuple(tupleDesc * tupleDesc, unsigned __int64 * values, bool * isnull, char * data, unsigned __int64 data_size, unsigned short * infomask, unsigned char * bit) Line 345	C
        // ������������ַ��������ȷ�����ݣ�������ַ��Ӧ������ �ο�HeapTupleHeaderData�ṹ�͹ٷ��ĵ���table 66.4��
        // char *data  td + hoff  ��Ӧ�������û����ݡ�
        // uint16 *infomask,   &td->t_infomask
        // bits8 *bit  td->t_bits   ��Ӧnull bitmap
 	postgres.exe!heap_form_tuple(tupleDesc * tupleDescriptor, unsigned __int64 * values, bool * isnull) Line 1157	C
        // �ڵ�ǰMemoryContext�����һ��HeapTuple��������tupleDescriptor�Ը��ֶε�������values�����ĸ��ֶε�ֵ������������õ�HeapTuple�С����HeapTuple���ڹ���buffer�У�֮��Ҫ������������������buffer�С�
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
            // ��������ҪŪ���Ҫ�������������α�ʾ�ģ�
            // �쿴 (ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree) �����ݡ�
            // ��һ������(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr�� ���е� funcid �ֶε�ֵ��1574
            // SELECT  oid, * FROM pg_proc where oid = 1574 ==> nextval
            
            // �ڶ������� (Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr�� ���е�consttype�ֶε�ֵ��1043
            // select oid, * from pg_type where oid = 1043; ==> varchar
            // ֪����varchar֮�������watch���� (char*)((Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr)->constvalue + 4 ����ַ���������ʲô�����ֹ�Ȼ��"foo"��
            
            // ((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist �������Ҫ���뵽���е����ݡ�
            
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```
postgresql��NodeTagΪ������������һ��"��̬"���ͻ��ơ�����ϸ���C���Է�������ת�����б�ʾ���ͻ��������watch���ʽһ�����Կ��������Ա��߷������������ָ����������Ķ��ķ�ʽ����������ʱ�����ݽṹ�����ֱ�ʾ��ʽû���ϸ�����ָ��ͽṹ��
�� "����1 =�ֶ�a=> ����2{ֵ}" ����ʾ ����1���ֶ�a����������������2��{}������������2��ֵ��
��� �ֶ�a��C����������List*����ô��  "=�ֶ�a[n]=>" ����ʽ����ʾ List*�ĵ�n��Ԫ�� �Ǽ�ͷ�ұߵ����͡�
"����1
     \=�ֶ�b=> ����2" ����ʽ ͬ����ʾ ����1���ֶ�b����������������2��

�������insertΪ�������ǿ��԰����ݽṹ������ֱ�۵���������:
                                         
Portal =stmts[0]=> (PlannedStmt*) =planTree=> (ModifyTable*) =plans[0]=> (Result*) =plan.targetlist[0]=> (TargetEntry*) =expr=> (FuncExpr*) =funcid=> Oid{1574}
                                                                                  \=plan.targetlist[1]=> (TargetEntry*) =expr=> (Const*) =consttype=> Oid{1043}
                                                                                                                                         \=constvalue=> Datum{"foo"}

ExecInitModifyTable ���� ModifyTableState -- �� ModifyTable��plans�е�Planִ��ExecInitNode�����ѵõ���PlanState����浽ModifyTableState��mt_plans�С�

ModifyTableState =ps.plan=> (ModifyTable*)
                 \=mt_plans[0]=> (ResultState*)
                 \=mt_nplans=> int (list_length(ModifyTable->plans))�� ����mt_plans���м���Ԫ�أ������������1��Ҳ����˵mt_plans��ֻ��һ��Ԫ��--ResultState
                 \=ps.ExecProcNode=> ExecModifyTable  

���������� ExecModifyTable ����εݹ���� ExecResult
```
>	postgres.exe!ExecResult(PlanState * pstate) Line 77	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecModifyTable(PlanState * pstate) Line 2027	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	������Ժ�֮ǰ��ͬ��stack��
```

    
```
>	postgres.exe!RelationPutHeapTuple(RelationData * relation, int buffer, HeapTupleData * tuple, bool token) Line 53	C
 	postgres.exe!heap_insert(RelationData * relation, HeapTupleData * tup, unsigned int cid, int options, BulkInsertStateData * bistate) Line 2490	C
        //���� heap_prepare_insert ����TOAST��
        // 
 	postgres.exe!ExecInsert(ModifyTableState * mtstate, TupleTableSlot * slot, TupleTableSlot * planSlot, EState * estate, bool canSetTag) Line 529	C
        // ���� heap_insert �� ExecMaterializeSlot �õ���tupleд��page��(����buffer)��
 	
    ������Ժ�֮ǰ��ͬ��stack��
```
    
Thus, a tuple is the latest version of its row iff XMAX is invalid or

### �ܽ�
ǰ����һ������������һ���������������չʾ��pg�Կͻ������ӣ�����buffer�������ڴ棬�����ļ�buffer�Ļ����÷���heap�Ļ�����֯�ṹ��Ҳ����pg��Ϊһ��C/S�ṹ�����ݴ�ȡ�������Ļ���ʵ�֡��ӵ����¿�ʼ������pg��Ϊһ����ϵ�����ݿ����ϵͳ�����������
