## 第三章 执行器 和 insert, delete, update的执行 ##

### 执行器简介
在postgres中，执行器(backend/executor)的工作是根据规划器(pg_plan_queries)返回的查询计划(Plan* PlannedStmt.planTree)执行查询(这里说的查询不仅包括读操作，也包括写操作，例如insert, delete, update)。
查询计划是个树形结构，这个树形结构中的每个节点被调用时都要返回下一个元组(tuple)，如果没有元组可返回，则返回NULL。如果一个节点有子节点，那么它要先获取子节点的元组，然后以这些子节点返回的元组作为数据源进行自己的运算后再返回--例如一个排序节点(struct Sort)要从它的子节点获取所有的元组然后排序，之后才能在每次被调用时从排好序的元组中返回下一个。 这样执行器执行树根的元素，其下各层子节点就会自动递归执行自己负责的部分，最终保证根节点的顺利执行。

查询计划对于执行器来说是个只读的数据结构。但是随着执行进度推进，势必会使当前这次"执行"的内部状态发生改变——例如执行扫描表的计划时要记录当前扫描到了哪条记录。postgres的做法是为查询计划树建立一个"平行"的查询状态树(PlanState*) -- 每一个查询计划树中的节点都有一个对应的查询状态树的结点，而且查询状态树的结点记录(plan字段)自己对应哪个查询计划树的结点。例如ModifyTable计划节点对应的状态节点是ModifyTableState。
不过有一种特殊情况: Expr 对应的状态节点 ExprState 不是个和Expr平行的树状结构，而是用一个数组(steps，数组元素的类型是ExprEvalStep)记录要计算的表达式，从Expr生成对应的ExprState的这部分工作由ExecInitExpr函数完成。这样做的目的在于: 
    1. 执行大多数Expr子节点的计算量比较小——比遍历树结构的开销大不了多少。不遍历树状结构就可以把计算能力更有效地用在处理用户需求(计算表达式)上面而不是用在内耗(遍历树结构)上。
    2. 避免递归调用，也就减少了栈深度和函数调用的次数。
    3. 平坦的数组结构既能支持快速的解释执行又能支持编译执行。
ExprState的执行过程大致是执行state->steps中的每个步骤(EEOP_ASSIGN_TMP，EEOP_CONST，EEOP_ASSIGN_TMP_MAKE_RO和EEOP_DONE类型的步骤 除外)，把计算结果放到state->resultslot->tts_values 和 state->resultslot->tts_isnull 中。EEOP_ASSIGN_TMP，EEOP_CONST和EEOP_ASSIGN_TMP_MAKE_RO类型的步骤负责把state->resultslot->tts_values 和 state->resultslot->tts_isnull 的值放到(ExprState* state)->resultslot对应的列上

把状态树作为一个和计划树独立的结构有一个重要的好处: 计划树在执行阶段是只读的，那么规划器可以很容易地维护查询语句和查询计划的cache。

除了对应计划树中的每个节点都有一个状态节点之外，还有一个EState结构记录本次执行器执行的全局状态。

执行器作为一个整体对postgres的其他模块提供了四个接口: ExecutorStart, ExecutorRun, ExecutorFinish 和 ExecutorEnd。 ProcessQuery 函数负责构建一个QueryDesc，并以QueryDesc为参数调用执行器的四个接口函数，上面提到的查询计划也是作为QueryDesc的plannedstmt字段传给执行器的。

QueryDesc =plannedstmt=> (PlannedStmt*) =planTree=> (Plan *)
ProcessQuery 函数封装了postgresql对执行器的调用:
```
QueryDesc  *queryDesc = CreateQueryDesc(...);
    
    
ExecutorStart(queryDesc, 0);   这个阶段的目的是构建和查询树平行的执行状态树。
    standard_ExecutorStart
        queryDesc->estate = CreateExecutorState();
        InitPlan(QueryDesc)  
            如果当前查询会向关系(relation)写数据，则要打开并以RowExclusiveLock模式锁住(heap_open)需要写的关系。这些关系记在plannedstmt->resultRelation里。
             plannedstmt->rowMarks  estate->es_rowMarks
            为每个子查询(plannedstmt->subplans)调用ExecInitNode构建状态树，并把构建好的状态树记到estate->es_subplanstates里。
            调用ExecInitNode为主查询构建状态树，并记到queryDesc->planstate里。
            ExecInitNode 根据不同的Plan(由NodeTag type字段标记)调用对应的 ExecInitXXXX 函数初始化对应的PlanState*。如果当前节点游子节点，那么ExecInitXXXX会对子节点递归调用ExecInitNode。
            
        
ExecutorRun(queryDesc, ForwardScanDirection, 0L, true) 对状态树的根节点(queryDesc->planstate)调用ExecutePlan。这个阶段是真正按查询树执行查询。
    standard_ExecutorRun
        ExecutePlan 
            ExecProcNode   
                node->ExecProcNode(node) 这个node是状态树的节点，不同类型的节点对应各自的ExecProcNode，是在上面提到的ExecInitNode中设置的。 例如 ModifyTableState 对应 ExecModifyTable。
        
设置 completionTag;
ExecutorFinish(queryDesc);
    standard_ExecutorFinish(queryDesc);
        执行after trigger。
ExecutorEnd(queryDesc);
    standard_ExecutorEnd(queryDesc);
        对执行状态树根节点调用ExecEndPlan。
        释放estate。
FreeQueryDesc(queryDesc);
    释放snapshots。
    释放queryDesc本身。
```

总结一下 —— postgresql里面不同查询的执行逻辑体现在不同 ExecXXX 函数中。对应查询状态树根节点的ExecXXX函数会递归调用子节点的ExecXXX函数。


### ExecModifyTable

本节后面要分析的insert, updata 和 delete语句在执行阶段都作为 ModifyTableState节点。它们都要调用子节点获取需要更改的数据，然后以这些数据为依据进行insert, update或delete。
ExecModifyTable 的大致框架如下:

```
调用before statement triggers

loop :
    // node.mt_whichplan 已经在 ExecInitModifyTable里面被初始化为0。
    对 node.mt_plans[node.mt_whichplan] 调用 ExecProcNode 获得下一行要改的数据 planSlot。
    if (planSlot == NULL) {
        // 当前node.mt_plans子节点已经没有进一步需要改的数据了。
        if (node.mt_whichplan < 最后一个节点) {
            // 让 node.mt_whichplan 指向下一个节点。
            node.mt_whichplan++;
        } else { 
            // node.mt_whichplan 已经是 node.mt_plans最后一个节点。走到这里表示已经没有进一步需要改的数据了。
            // 退出循环。
            break;
        }
    }
    
    if (正在做update或delete) {
        ExecGetJunkAttribute从planSlot中提取ctid。ctid就是要被update或者delete的row的"物理"位置。
    }
    对 planSlot 调用 ExecInsert, ExecUpdate 或者 ExecDelete。得到实际改变后的数据slot。
    if ( slot!=NULL ) {
        // 查询中带有RETURNNING子句。
        return slot;
    }
}

```

之后分别查看insert, delete和update具体是如何实现的。


### insert

```
insert into table1(col1) values ('foo');
```

ExecInitNode 阶段:
```
    postgres.exe!ExecInitFunc(ExprEvalStep * scratch, Expr * node, List * args, unsigned int funcid, unsigned int inputcollid, ExprState * state) Line 2159	C
        初始化调用函数的结点，在我们这个例子，要调用的函数是nextval。
 	postgres.exe!ExecInitExprRec(Expr * node, ExprState * state, unsigned __int64 * resv, bool * resnull) Line 885	C
        根据node->type构造对应的ExprEvalStep，并且用ExprEvalPushStep把构造出来的ExprEvalStep放到state->steps数组里。
        子节点有可能递归调用ExecInitExprRec，例如ExecInitFunc会对每个参数递归调用ExecInitExprRec。
        注意这里有:
        scratch.resvalue = resv;
        scratch.resnull = resnull;
        给scratch(scratch就是要得到的step)指定了把计算结果存到哪个地址。因为ExecBuildProjectionInfo给定了&state->resvalue和&state->resnull，所以scratch会把计算结果放到&state->resvalue和&state->resnull。
        
 	postgres.exe!ExecBuildProjectionInfo(List * targetList, ExprContext * econtext, TupleTableSlot * slot, PlanState * parent, tupleDesc * inputDesc) Line 459	C
        execExpr.c:349
        这个函数的作用是构建一个ProjectionInfo *projInfo。
        ExecInitExprSlots(state, (Node *) targetList);
        
        ProjectionInfo.pi_state的类型是ExprState，这里要对
        对targetlist里的每一个TargetEntry 调用 ExecInitExprRec，还要在每次调用ExecInitExprRec后向ProjectionInfo.pi_state.steps数组里 增加一个 EEOP_ASSIGN_TMP或者EEOP_ASSIGN_TMP_MAKE_RO操作。
        EEOP_ASSIGN_TMP或者EEOP_ASSIGN_TMP_MAKE_RO操作的作用是把 ExprState的计算结果(resvalue/resnull)放到(ExprState* state)->resultslot对应的列上。
        这样，ProjectionInfo.pi_state.steps里面就包含了要得到一个tuple所需要的所有步骤。在执行阶段，顺序执行其中的每个步骤之后，就填好了tuple的每个字段。
        
        所以在这里为Result节点的targetlist中的元素建立的ExprState和ResultState节点的关系是: ResultState.(PlanState)ps.(ProjectionInfo *)ps_ProjInfo->(ExprState)pi_state
        
 	postgres.exe!ExecAssignProjectionInfo(PlanState * planstate, tupleDesc * inputDesc) Line 467	C
        用 ExecBuildProjectionInfo 初始化 planstate->ps_ProjInfo
        planstate->ps_ResultTupleSlot
        
>	postgres.exe!ExecInitResult(Result * node, EState * estate, int eflags) Line 226	C
        调用ExecInitResultTupleSlotTL初始化planstate->ps_ResultTupleSlot
        
 	postgres.exe!ExecInitNode(Plan * node, EState * estate, int eflags) Line 164	C
 	postgres.exe!ExecInitModifyTable(ModifyTable * node, EState * estate, int eflags) Line 2306	C
 	postgres.exe!ExecInitNode(Plan * node, EState * estate, int eflags) Line 174	C
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
        // 察看 (ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree) 的内容。
        // 第一个参数(FuncExpr*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->data.ptr_value)->expr， 其中的 funcid 字段的值是1574
        // SELECT  oid, * FROM pg_proc where oid = 1574 ==> nextval
        
        // 第二个参数 (Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr， 其中的consttype字段的值是1043
        // select oid, * from pg_type where oid = 1043; ==> varchar
        // 知道是varchar之后，用这个watch看看 (char*)((Const*)((TargetEntry*)((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist->head->next->data.ptr_value)->expr)->constvalue + 4 这个字符串到底是什么，发现果然是"foo"。
        
        // ((Result*)((ModifyTable*)(((PlannedStmt *)(portal->stmts->head->data.ptr_value))->planTree))->plans->head->data.ptr_value)->plan.targetlist 用来存放要插入到表中的数据。
        
        然后是插入的目标表:
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable 里面记录着这个查询所可能用到的所有的表，在我们这个例子中只用到了一个表:
        ((RangeTblEntry*)((PlannedStmt *)(portal->stmts->head->data.ptr_value))->rtable->head->data.ptr_value)->relid 和 SELECT oid, * FROM pg_catalog.pg_class where relname='table1'; 得到oid正好吻合。
        
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations 里面记录着insert的目标表在rttable中的索引，用这个watch可以看到它的值是1:
        // ((PlannedStmt *)(portal->stmts->head->data.ptr_value))->resultRelations->head->data.int_value
    
 	postgres.exe!PostgresMain(int argc, char * * argv, const char * dbname, const char * username) Line 4155	C
 	postgres.exe!BackendRun(Port * port) Line 4362	C
 	postgres.exe!SubPostmasterMain(int argc, char * * argv) Line 4885	C
 	postgres.exe!main(int argc, char * * argv) Line 216	C

```

在我们这个简单的例子中，Result的targetlist中只有两种类型的子节点，函数(T_FuncExpr，而且nextval是个无参数的函数)和常数(T_Const)。函数的情况在上面已经看过了，现在来看常量的情况:
```
>	postgres.exe!ExecInitExprRec(Expr * node, ExprState * state, unsigned __int64 * resv, bool * resnull) Line 720	C
            case T_Const:
			{
				Const	   *con = (Const *) node;

				scratch.opcode = EEOP_CONST;
				scratch.d.constval.value = con->constvalue;  <== 这个就是常量字符串"foo"。
				scratch.d.constval.isnull = con->constisnull;
				ExprEvalPushStep(state, &scratch);
				break;
			}
            在这里用datumGetSize(scratch.d.constval.value, 0, -1)可以看到数据的总长度是7，7=4(varlena.vl_len_本身的长度 )+3("foo"的长度)
 	postgres.exe!ExecBuildProjectionInfo(List * targetList, ExprContext * econtext, TupleTableSlot * slot, PlanState * parent, tupleDesc * inputDesc) Line 459	C
    
```
    
    

ExecBuildProjectionInfo当返回之前，state->steps中有5个step(step_lens=5)
steps[0] opcode=EEOP_FUNCEXPR_STRICT(18), op->d.func.finfo记录要调用的函数的信息，其中op->d.func.finfo.fn_oid字段对应系统表pg_catalog.pg_proc的oid字段
    ```
    EEO_CASE(EEOP_FUNCEXPR_STRICT)
		{
			FunctionCallInfo fcinfo = op->d.func.fcinfo_data;
			bool	   *argnull = fcinfo->argnull;
			int			argno;
			Datum		d;

			/* strict function, so check for NULL args */
			for (argno = 0; argno < op->d.func.nargs; argno++)
			{
				if (argnull[argno])
				{
					*op->resnull = true;
					goto strictfail;
				}
			}
			fcinfo->isnull = false;
			d = op->d.func.fn_addr(fcinfo); //  d.func.fn_addr是op->d.func.finfo.fn_oid对应的C函数的地址。
			*op->resvalue = d;              // 之前已经说明过op->resvalue和state->resvalue是同一个地址，相当于赋值给了state->resvalue。  
			*op->resnull = fcinfo->isnull;  // 之前已经说明过op->resnull和state->resnull是同一个地址，相当于赋值给了state->resnull。
            // 因为后面的EEOP_ASSIGN_TMP只能把state里面的数据赋值到resultslot->tts_XXX里面，这里把op->resXXX指向state->resXXX。
            // 不能直接 state->resvalue = d 和 state->resnull = fcinfo->isnull，是因为在其他情况下有可能需要把结果存到其他地方而不是state上。
            
	strictfail:
			EEO_NEXT();
		}
     ```
steps[1] opcode=EEOP_ASSIGN_TMP(14)，
    把上一步的计算结果赋值到resultslot->tts_values和resultslot->tts_isnull 相应的位置，位置由d.assign_tmp.resultnum给出。
    ```
    EEO_CASE(EEOP_ASSIGN_TMP)
		{
			int			resultnum = op->d.assign_tmp.resultnum;

			resultslot->tts_values[resultnum] = state->resvalue;
			resultslot->tts_isnull[resultnum] = state->resnull;

			EEO_NEXT();
		}
    ```

steps[2] opcode=EEOP_CONST(16)，
    把op->d.constval复制给state->resvalue和state->resnull，以便下一步EEOP_ASSIGN_TMP_MAKE_RO能把这个常量复制给resultslot。
    ```
    EEO_CASE(EEOP_CONST)
		{
			*op->resnull = op->d.constval.isnull;
			*op->resvalue = op->d.constval.value;

			EEO_NEXT();
		}
    ```
steps[3] opcode=EEOP_ASSIGN_TMP_MAKE_RO(15)，
    ```
    EEO_CASE(EEOP_ASSIGN_TMP_MAKE_RO)
		{
			int			resultnum = op->d.assign_tmp.resultnum;

			resultslot->tts_isnull[resultnum] = state->resnull;
			if (!resultslot->tts_isnull[resultnum])
				resultslot->tts_values[resultnum] =
					MakeExpandedObjectReadOnlyInternal(state->resvalue); //只是比EEOP_ASSIGN_TMP多了这个步骤。
			else
				resultslot->tts_values[resultnum] = state->resvalue;

			EEO_NEXT();
		}
    ```
steps[4] opcode=EEOP_DONE  表示这是最后一个step。
    ```
    EEO_CASE(EEOP_DONE)
		{
			goto out;  结束ExecInterpExpr函数。
		}
    ```
step的opcode除了上面5个类型之外，完整的列表 enum ExprEvalOp 定义在 execExpr.h 。

ExecProcNode 阶段

insert的执行阶段可以大致分成三个步骤:
一、对node->mt_plans[node->mt_whichplan]调用ExecProcNode，得到要插入的数据slot。在我们这个例子里，node是个ModifyTableState*，node->mt_plans只有一个元素，这个元素是个ResultState*
二、对slot调用ExecMaterializeSlot得到可以写到数据页(heap page)里的形式tuple。
三、用heap_insert把tuple写到数据页里。

下面分别来分析这三个步骤:
一、得到TupleTableSlot* slot:

这需要调用 ExecResult
```
>	postgres.exe!ExecInterpExpr(ExprState * state, ExprContext * econtext, bool * isnull) Line 423	C
        执行state->steps中的每个步骤，把计算结果放到state->resultslot->tts_values 和 state->resultslot->tts_isnull 中对应的位置。
 	postgres.exe!ExecInterpExprStillValid(ExprState * state, ExprContext * econtext, bool * isNull) Line 1787	C
 	postgres.exe!ExecEvalExprSwitchContext(ExprState * state, ExprContext * econtext, bool * isNull) Line 303	C
 	postgres.exe!ExecProject(ProjectionInfo * projInfo) Line 343	C
 	postgres.exe!ExecResult(PlanState * pstate) Line 136	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecModifyTable(PlanState * pstate) Line 2027	C
 	postgres.exe!ExecProcNodeFirst(PlanState * node) Line 446	C
 	postgres.exe!ExecProcNode(PlanState * node) Line 238	C
 	postgres.exe!ExecutePlan(EState * estate, PlanState * planstate, bool use_parallel_mode, CmdType operation, bool sendTuples, unsigned __int64 numberTuples, ScanDirection direction, _DestReceiver * dest, bool execute_once) Line 1721	C
 	postgres.exe!standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 376	C
 	postgres.exe!ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, unsigned __int64 count, bool execute_once) Line 306	C
 	postgres.exe!ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfoData * params, QueryEnvironment * queryEnv, _DestReceiver * dest, char * completionTag) Line 166	C
 	...省略和之前相同的栈。 
```



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
 	...省略和之前相同的栈。 
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
                 \=mt_nplans=> int (list_length(ModifyTable->plans))， 数组mt_plans中有元素个数，在这个例子中是1，也就是说mt_plans中只有一个元素--ResultState
                 \=ps.ExecProcNode=> ExecModifyTable  


    
```
>	postgres.exe!RelationPutHeapTuple(RelationData * relation, int buffer, HeapTupleData * tuple, bool token) Line 53	C
 	postgres.exe!heap_insert(RelationData * relation, HeapTupleData * tup, unsigned int cid, int options, BulkInsertStateData * bistate) Line 2490	C
        //调用 heap_prepare_insert 处理TOAST。
        // 
 	postgres.exe!ExecInsert(ModifyTableState * mtstate, TupleTableSlot * slot, TupleTableSlot * planSlot, EState * estate, bool canSetTag) Line 529	C
        // 调用 heap_insert 把 ExecMaterializeSlot 得到的tuple写到page里(共享buffer)。
 	
    ...省略和之前相同的栈。 
```
    
Thus, a tuple is the latest version of its row iff XMAX is invalid or

### 总结
前三章一起给读者描绘了一个轮廓，这个轮廓展示了pg对客户端连接，网络buffer，共享内存，数据文件buffer的基本用法和heap的基本组织结构。也就是pg作为一个C/S结构的数据存取服务器的基本实现。从第四章开始，将对pg作为一个关系型数据库管理系统做深入分析。
