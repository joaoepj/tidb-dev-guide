# Debug and profile

In this section, you will learn:

* How to debug TiDB
* How to pause the execution at any line of code to inspect values and stacks
* How to profile TiDB to catch a performance bottleneck

## Use delve for debugging

[Delve](https://github.com/go-delve/delve) is a debugger for the Go programming language. It provides a command-line debugging experience similar to the GNU Project debugger (GDB), but it is much more Go native than GDB itself.

### Install delve

To install delve, see the [installation guide](https://github.com/go-delve/delve/tree/master/Documentation/installation). After the installation, depending on how you set your environment variables, you will have an executable file named `dlv` in either `$GOPATH/bin` or `$HOME/go/bin`. You can then run the following command to verify the installation:

```
$ dlv version
Delve Debugger
Version: 1.5.0
Build: $Id: ca5318932770ca063fc9885b4764c30bfaf8a199 $
```

### Attach delve to a running TiDB process

Once you get the TiDB server running, you can attach the delve debugger.

For example, you can build and run a standalone TiDB server by running the following commands in the root directory of the source code:

```bash
make server
./bin/tidb-server
```

You can then start a new shell and use `ps` or `pgrep` to find the PID of the tidb server process you just started:

```bash
pgrep tidb-server
# OUTPUT:
# 1394942
```

If the output lists multiple PIDs, it indicates that you might have multiple TiDB servers running at the same time. To determine the PID of the tidb server you are planning to debug, you can use commands such as `ps $PID`, where `$PID` is the PID you are trying to know more about:

```bash
ps 1394942
# OUTPUT:
#     PID TTY      STAT   TIME COMMAND
# 1394942 pts/11   SNl    0:02 ./bin/tidb-server
```

Once you get the PID, you can attach delve to it by running the following command:

```bash
dlv attach 1394942
```

You might get error messages of the kernel security setting as follows:

```
Could not attach to pid 1394942: this could be caused by a kernel security setting, try writing "0" to /proc/sys/kernel/yama/ptrace_scope
```

To resolve the error, follow the instructions provided in the error message and execute the following command as the root user to override the kernel security setting:

```bash
echo 0 > /proc/sys/kernel/yama/ptrace_scope
```

Then retry attaching delve onto the PID, and it should work.

If you've worked with GDB, the delve debugging interface will look familiar to you. It is an interactive dialogue that allows you to interact with the execution of the tidb server attached on. To learn more about delve, you can type help into the dialogue and read the `help` messages.

### Setting breakpoints

After attaching delve to the running TiDB server process, you can now set breakpoints. TiDB server will pause execution at the breakpoints you specify.

To create a breakpoint, you can write:

```
break [name] <linespec>
```

where `[name]` is the name for the breakpoint, and `<linespec>` is the position of a line of code in the source code. Note the name is optional.

For example, the following command creates a breakpoint at the `Next` function of `HashJoinExec`. (The line number can be subject to change due to the modification of the source code).

```bash
dlv debug tidb-server/main.go
# OUTPUT:
# Type 'help' for list of commands.
# (dlv) break executor/join.go:653
# Breakpoint 1 (enabled) set at 0x36752d8 for github.com/pingcap/tidb/executor.(*HashJoinExec).Next() ./executor/join.go:653
# (dlv)
```

Once the execution is paused, the context of the execution is fully preserved. You are free to inspect the values of different variables, print the calling stack, and even jump between different goroutines. Once you finish the inspection, you can resume the execution by stepping into the next line of code or continue the execution until the next breakpoint is encountered.

Typically, when you use a debugger, you need to take the following steps:

1. Locate the code and set a breakpoint.
2. Prepare data so that the execution will get through the breakpoint, and pause at the specified breakpoint as expected.
3. Inspect values and follow the execution step by step.



### Debugging a test case

If a test case fails, you can also use delve to debug it. Get the name of the test case, go to the corresponding package directory, and then run the following command to start a debugging session that will stop at the entry of the test:

```
dlv test -- -run TestName
```

### Understand how TiDB works through debugging

Besides debugging problems, you can also use the debugger to understand how TiDB works through tracking the execution step by step.

To understand TiDB internals, it's critical that you understand certain functions. To better understand how TiDB works, you can pause the execution of these TiDB functions, and then run TiDB step by step.

For example:

1. [`executor/compiler.go:Compile`](https://github.com/pingcap/tidb/blob/5c95062cc34d6d37e2e921f9bddba6205b43ee3a/executor/compiler.go#L48) is where each SQL statement is compiled and optimized.
2. [`planner/planner.go:Optimize`](https://github.com/pingcap/tidb/blob/5c95062cc34d6d37e2e921f9bddba6205b43ee3a/planner/optimize.go#L80) is where the SQL optimization starts.
3. [`executor/adapter.go:ExecStmt.Exec`](https://github.com/pingcap/tidb/blob/5c95062cc34d6d37e2e921f9bddba6205b43ee3a/executor/adapter.go#L312) is where the SQL plan turns into executor and where the SQL execution starts.
4. Each executor's `Open`, `Next`, and `Close` function marks the volcano-style execution logic.

When you are reading the TiDB source code, you are strongly encouraged to set a breakpoint and use the debugger to trace the execution whenever you are confused or uncertain about the code.

### Tracking the execution of SELECT statements
For example, you can use the debugger to track the execution of simple SELECT statements.

Start the TiDB server, attach the debugger to it as instructed in the previous section and connect a client to it.

#### Parsing the string

At debugger, set a breakpoint to the handleQuery function with command `break server.(*clientConn).handleQuery` and continue the execution with `continue`.

[`server/conn.go:handleQuery`](https://github.com/pingcap/tidb/blob/release-6.5/server/conn.go#L1877)

Get back to the client and input the statement `SELECT 1;`. Note that the prompt doesn't return indicating that the execution stopped at the breakpoint.

#### Inspecting the AST

At debugger, you should now see some lines of code from handleQuery function. Type `next` and then press `<ENTER>` until the variable `stmt` is instantiated. At this point, you should be able to see `stmt` contents and typing `print stmt` should result in something like the output below.

Been acquainted with golang concepts like interfaces and embedding will be helpful during your tracking sessions. In addition, reading the definition of the types referenced in the output during the sessions also has its value. For now, you can just ignore every line terminating in `nil,`.

Delve prints interfaces using the syntax _\<interface name\>(\<concrete type\>) \<value\>_. This means that the structure represented by _\<value\>_ has the type _\<concrete type\>_ which implements the interface _\<interface name\>_. In the first line of the output you can see that the variable `stmt` holds a pointer to a `StmtSelect struct` which implements the `StmtNode interface`.

Delve prints embedded elements like interfaces and structs. For example, output's second line shows `dmlNode` as a field of the struct with the type struct `ast.dmlNode`. For your turn, this inner struct also embeds another struct of type `stmtNode`.

When delve evaluates a memory address it will automatically return the value of nested struct members, array and slice items and dereference pointers. However to limit the size of the output evaluation will be limited to two levels deep. Beyond two levels only the address of the item will be returned as you can see in the line highlithed in blue.

```
github.com/pingcap/tidb/parser/ast.StmtNode(*github.com/pingcap/tidb/parser/ast.SelectStmt) *{
        dmlNode: github.com/pingcap/tidb/parser/ast.dmlNode {
                stmtNode: (*"github.com/pingcap/tidb/parser/ast.stmtNode")(0xc015666120),},
        SelectStmtOpts: *github.com/pingcap/tidb/parser/ast.SelectStmtOpts {
                Distinct: false,
                SQLBigResult: false,
                SQLBufferResult: false,
                SQLCache: true,
                SQLSmallResult: false,
                CalcFoundRows: false,
                StraightJoin: false,
                Priority: NoPriority (0),
                TableHints: []*github.com/pingcap/tidb/parser/ast.TableOptimizerHint len: 0, cap: 0, nil,
                ExplicitAll: false,},
        Distinct: false,
        From: *github.com/pingcap/tidb/parser/ast.TableRefsClause nil,
        Where: github.com/pingcap/tidb/parser/ast.ExprNode nil,
        Fields: *github.com/pingcap/tidb/parser/ast.FieldList {
                node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc0115ec000),
                Fields: []*github.com/pingcap/tidb/parser/ast.SelectField len: 1, cap: 1, [
                        *(*"github.com/pingcap/tidb/parser/ast.SelectField")(0xc01716c120),
                ],},
        GroupBy: *github.com/pingcap/tidb/parser/ast.GroupByClause nil,
        Having: *github.com/pingcap/tidb/parser/ast.HavingClause nil,
        WindowSpecs: []github.com/pingcap/tidb/parser/ast.WindowSpec len: 0, cap: 0, nil,
        OrderBy: *github.com/pingcap/tidb/parser/ast.OrderByClause nil,
        Limit: *github.com/pingcap/tidb/parser/ast.Limit nil,
        LockInfo: *github.com/pingcap/tidb/parser/ast.SelectLockInfo nil,
        TableHints: []*github.com/pingcap/tidb/parser/ast.TableOptimizerHint len: 0, cap: 0, nil,
        IsInBraces: false,
        WithBeforeBraces: false,
        QueryBlockOffset: 0,
        SelectIntoOpt: *github.com/pingcap/tidb/parser/ast.SelectIntoOption nil,
        AfterSetOperator: *github.com/pingcap/tidb/parser/ast.SetOprType nil,
        Kind: SelectStmtKindSelect (0),
        Lists: []*github.com/pingcap/tidb/parser/ast.RowExpr len: 0, cap: 0, nil,
        With: *github.com/pingcap/tidb/parser/ast.WithClause nil,
        AsViewSchema: false,}
```

#### Shedding some light on ASTs

The select field list:

```
(dlv) p stmtNode.Fields
*github.com/pingcap/tidb/parser/ast.FieldList {
        node: github.com/pingcap/tidb/parser/ast.node {
                utf8Text: "",
                enc: github.com/pingcap/tidb/parser/charset.Encoding nil,
                once: *sync.Once nil,
                text: "",
                offset: 0,},
        Fields: []*github.com/pingcap/tidb/parser/ast.SelectField len: 2, cap: 2, [
                *(*"github.com/pingcap/tidb/parser/ast.SelectField")(0xc0141199e0),
                *(*"github.com/pingcap/tidb/parser/ast.SelectField")(0xc014119a70),
        ],}
```

The first and only field:

```
(dlv) p stmtNode.Fields.Fields[0]
*github.com/pingcap/tidb/parser/ast.SelectField {
        node: github.com/pingcap/tidb/parser/ast.node {
                utf8Text: "1+1",
                enc: github.com/pingcap/tidb/parser/charset.Encoding(*github.com/pingcap/tidb/parser/charset.encodingUTF8) ...,
                once: *(*sync.Once)(0xc012d291a0),
                text: "1+1",
                offset: 0,},
        Offset: 7,
        WildCard: *github.com/pingcap/tidb/parser/ast.WildCardField nil,
        Expr: github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/parser/ast.BinaryOperationExpr) *{
                exprNode: (*"github.com/pingcap/tidb/parser/ast.exprNode")(0xc012df9420),
                Op: Plus (11),
                L: github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) ...,
                R: github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) ...,},
        AsName: github.com/pingcap/tidb/parser/model.CIStr {O: "", L: ""},
        Auxiliary: false,
        AuxiliaryColInAgg: false,
        AuxiliaryColInOrderBy: false,}
```

The field is a binary operation expression:

```
(dlv) p stmtNode.Fields.Fields[0].Expr
github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/parser/ast.BinaryOperationExpr) *{
        exprNode: github.com/pingcap/tidb/parser/ast.exprNode {
                node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc012df9420),
                Type: (*"github.com/pingcap/tidb/parser/types.FieldType")(0xc012df9460),
                flag: 0,},
        Op: Plus (11),
        L: github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) *{
                TexprNode: (*"github.com/pingcap/tidb/parser/ast.exprNode")(0xc012bc78c0),
                Datum: (*"github.com/pingcap/tidb/types.Datum")(0xc012bc7978),
                projectionOffset: -1,},
        R: github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) *{
                TexprNode: (*"github.com/pingcap/tidb/parser/ast.exprNode")(0xc012bc79e0),
                Datum: (*"github.com/pingcap/tidb/types.Datum")(0xc012bc7a98),
                projectionOffset: -1,},}
```
The left side of the expression:

```
(dlv) p stmtNode.Fields.Fields[0].Expr.L
github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) *{
        TexprNode: github.com/pingcap/tidb/parser/ast.exprNode {
                node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc012bc78c0),
                Type: (*"github.com/pingcap/tidb/parser/types.FieldType")(0xc012bc7900),
                flag: 0,},
        Datum: github.com/pingcap/tidb/types.Datum {
                k: 1,
                decimal: 0,
                length: 0,
                i: 1,
                collation: "",
                b: []uint8 len: 0, cap: 0, nil,
                x: interface {} nil,},
        projectionOffset: -1,}
```

The binary operator:

```
(dlv) p stmtNode.Fields.Fields[0].Expr.Op
Plus (11)
(dlv) p stmtNode.Fields.Fields[0].Expr.L
github.com/pingcap/tidb/parser/ast.ExprNode(*github.com/pingcap/tidb/types/parser_driver.ValueExpr) *{
        TexprNode: github.com/pingcap/tidb/parser/ast.exprNode {
                node: (*"github.com/pingcap/tidb/parser/ast.node")(0xc012bc78c0),
                Type: (*"github.com/pingcap/tidb/parser/types.FieldType")(0xc012bc7900),
                flag: 0,},
        Datum: github.com/pingcap/tidb/types.Datum {
                k: 1,
                decimal: 0,
                length: 0,
                i: 1,
                collation: "",
                b: []uint8 len: 0, cap: 0, nil,
                x: interface {} nil,},
        projectionOffset: -1,}
```


#### Logging complex structures

Start debugging session from here using delve text mode:

```
break /home/tidb/session/session.go:2167
condition 1 stmt.Text == "SELECT 1+1"
```

Defined in exectutor/adapter.go:233
```
// ExecStmt implements the sqlexec.Statement interface, it builds a planner.Plan to an sqlexec.Statement.
type ExecStmt struct {
	// GoCtx stores parent go context.Context for a stmt.
	GoCtx context.Context
	// InfoSchema stores a reference to the schema information.
	InfoSchema infoschema.InfoSchema
	// Plan stores a reference to the final physical plan.
	Plan plannercore.Plan
	// Text represents the origin query text.
	Text string

	StmtNode ast.StmtNode

	Ctx sessionctx.Context

	// LowerPriority represents whether to lower the execution priority of a query.
	LowerPriority     bool
	isPreparedStmt    bool
	isSelectForUpdate bool
	retryCount        uint
	retryStartTime    time.Time

	// Phase durations are splited into two parts: 1. trying to lock keys (but
	// failed); 2. the final iteration of the retry loop. Here we use
	// [2]time.Duration to record such info for each phase. The first duration
	// is increased only within the current iteration. When we meet a
	// pessimistic lock error and decide to retry, we add the first duration to
	// the second and reset the first to 0 by calling `resetPhaseDurations`.
	phaseBuildDurations [2]time.Duration
	phaseOpenDurations  [2]time.Duration
	phaseNextDurations  [2]time.Duration
	phaseLockDurations  [2]time.Duration

	// OutputNames will be set if using cached plan
	OutputNames []*types.FieldName
	PsStmt      *plannercore.PlanCacheStmt
	Ti          *TelemetryInfo
}
```

This line of code:
```
logutil.Logger(ctx).Info("var stmt *executor.ExecStmt", zap.Any("stmt", stmt))
```

Printing stmt on dlv prompt seems much more pretty, compare with the output below:

```
*github.com/pingcap/tidb/executor.ExecStmt {
        GoCtx: context.Context(*context.valueCtx) *{
                Context: context.Context(*context.valueCtx) ...,
                key: (unreadable could not resolve interface type),
                val: interface {}(*github.com/tikv/client-go/v2/util.ExecDetails) ...,},
        InfoSchema: github.com/pingcap/tidb/infoschema.InfoSchema(*github.com/pingcap/tidb/infoschema.SessionExtendedInfoSchema) *{
                InfoSchema: github.com/pingcap/tidb/infoschema.InfoSchema(*github.com/pingcap/tidb/infoschema.infoSchema) ...,
                LocalTemporaryTablesOnce: (*sync.Once)(0xc012151240),
                LocalTemporaryTables: *github.com/pingcap/tidb/infoschema.SessionTables nil,
                MdlTables: *github.com/pingcap/tidb/infoschema.SessionTables nil,},
        Plan: github.com/pingcap/tidb/planner/core.Plan(*github.com/pingcap/tidb/planner/core.PhysicalProjection) *{
                physicalSchemaProducer: (*"github.com/pingcap/tidb/planner/core.physicalSchemaProducer")(0xc011d7f0a0),
                Exprs: []github.com/pingcap/tidb/expression.Expression len: 1, cap: 1, [
                        ...,
                ],
                CalculateNoDelay: false,
                AvoidColumnEvaluator: false,},
        Text: "SELECT 1+1",
        StmtNode: github.com/pingcap/tidb/parser/ast.StmtNode(*github.com/pingcap/tidb/parser/ast.SelectStmt) *{
                dmlNode: (*"github.com/pingcap/tidb/parser/ast.dmlNode")(0xc0120d0ea0),
                SelectStmtOpts: *(*"github.com/pingcap/tidb/parser/ast.SelectStmtOpts")(0xc0121511a0),
                Distinct: false,
                From: *github.com/pingcap/tidb/parser/ast.TableRefsClause nil,
                Where: github.com/pingcap/tidb/parser/ast.ExprNode nil,
                Fields: *(*"github.com/pingcap/tidb/parser/ast.FieldList")(0xc0120e2840),
                GroupBy: *github.com/pingcap/tidb/parser/ast.GroupByClause nil,
                Having: *github.com/pingcap/tidb/parser/ast.HavingClause nil,
                WindowSpecs: []github.com/pingcap/tidb/parser/ast.WindowSpec len: 0, cap: 0, nil,
                OrderBy: *github.com/pingcap/tidb/parser/ast.OrderByClause nil,
                Limit: *github.com/pingcap/tidb/parser/ast.Limit nil,
                LockInfo: *github.com/pingcap/tidb/parser/ast.SelectLockInfo nil,
                TableHints: []*github.com/pingcap/tidb/parser/ast.TableOptimizerHint len: 0, cap: 0, nil,
                IsInBraces: false,
                WithBeforeBraces: false,
                QueryBlockOffset: 1,
                SelectIntoOpt: *github.com/pingcap/tidb/parser/ast.SelectIntoOption nil,
                AfterSetOperator: *github.com/pingcap/tidb/parser/ast.SetOprType nil,
                Kind: SelectStmtKindSelect (0),
                Lists: []*github.com/pingcap/tidb/parser/ast.RowExpr len: 0, cap: 0, nil,
                With: *github.com/pingcap/tidb/parser/ast.WithClause nil,
                AsViewSchema: false,},
        Ctx: github.com/pingcap/tidb/sessionctx.Context(*github.com/pingcap/tidb/session.session) *{
                processInfo: (*"sync/atomic.Value")(0xc0113b3400),
                txn: (*"github.com/pingcap/tidb/session.LazyTxn")(0xc0113b3410),
                mu: (*"struct { sync.RWMutex; github.com/pingcap/tidb/session.values map[fmt.Stringer]interface {} }")(0xc0113b3538),
                currentCtx: context.Context(*context.valueCtx) ...,
                currentPlan: github.com/pingcap/tidb/planner/core.Plan nil,
                store: github.com/pingcap/tidb/kv.Storage(*github.com/pingcap/tidb/store/mockstore/mockstorage.mockStorage) ...,
                preparedPlanCache: github.com/pingcap/tidb/sessionctx.PlanCache nil,
                generalPlanCache: github.com/pingcap/tidb/sessionctx.PlanCache nil,
                sessionVars: *(*"github.com/pingcap/tidb/sessionctx/variable.SessionVars")(0xc011375800),
                sessionManager: github.com/pingcap/tidb/util.SessionManager(*github.com/pingcap/tidb/server.Server) ...,
                statsCollector: *(*"github.com/pingcap/tidb/statistics/handle.SessionStatsCollector")(0xc01138f840),
                ddlOwnerManager: github.com/pingcap/tidb/owner.Manager(*github.com/pingcap/tidb/owner.mockManager) ...,
                lockedTables: map[int64]github.com/pingcap/tidb/parser/model.TableLockTpInfo [],
                client: github.com/pingcap/tidb/kv.Client(*github.com/pingcap/tidb/store/copr.CopClient) ...,
                mppClient: github.com/pingcap/tidb/kv.MPPClient(*github.com/pingcap/tidb/store/copr.MPPClient) ...,
                idxUsageCollector: *github.com/pingcap/tidb/statistics/handle.SessionIndexUsageCollector nil,
                functionUsageMu: (*"struct { sync.RWMutex; github.com/pingcap/tidb/session.builtinFunctionUsage github.com/pingcap/tidb/telemetry.BuiltinFunctionsUsage }")(0xc0113b3608),
                diskFullOpt: DiskFullOpt_NotAllowedOnFull (0),
                stmtStats: *(*"github.com/pingcap/tidb/util/topsql/stmtstats.StatementStats")(0xc010ab3938),
                sessionStatesHandlers: map[github.com/pingcap/tidb/sessionctx/sessionstates.SessionStateType]github.com/pingcap/tidb/sessionctx.SessionStatesHandler [...],
                advisoryLocks: map[string]*github.com/pingcap/tidb/session.advisoryLock [],
                extensions: *github.com/pingcap/tidb/extension.SessionExtensions nil,
                sandBoxMode: false,},
        LowerPriority: false,
        isPreparedStmt: false,
        isSelectForUpdate: false,
        retryCount: 0,
        retryStartTime: time.Time(0001-01-01T00:00:00Z){
                wall: 0,
                ext: 0,
                loc: *time.Location nil,},
        phaseBuildDurations: [2]time.Duration [github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0),github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0)],
        phaseOpenDurations: [2]time.Duration [github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0),github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0)],
        phaseNextDurations: [2]time.Duration [github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0),github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0)],
        phaseLockDurations: [2]time.Duration [github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0),github.com/tikv/pd/client.defaultMaxTSOBatchWaitInterval (0)],
        OutputNames: []*github.com/pingcap/tidb/types.FieldName len: 1, cap: 1, [
                *(*"github.com/pingcap/tidb/types.FieldName")(0xc012118e70),
        ],
        PsStmt: *github.com/pingcap/tidb/planner/core.PlanCacheStmt nil,
        Ti: *github.com/pingcap/tidb/executor.TelemetryInfo {
                UseNonRecursive: false,
                UseRecursive: false,
                UseMultiSchemaChange: false,
                UesExchangePartition: false,
                UseFlashbackToCluster: false,
                PartitionTelemetry: *github.com/pingcap/tidb/executor.PartitionTelemetryInfo nil,
                AccountLockTelemetry: *github.com/pingcap/tidb/executor.AccountLockTelemetryInfo nil,
                UseIndexMerge: false,},}
```



Prints like this on log:
```
[2023/01/12 10:19:52.591 -03:00] [INFO] [session.go:2169] ["var stmt *executor.ExecStmt"] [stmt="{\"GoCtx\":{\"Context\":0},\"InfoSchema\":{\"InfoSchema\":{},\"LocalTemporaryTablesOnce\":{},\"LocalTemporaryTables\":null,\"MdlTables\":null},\"Plan\":{\"TiFlashFineGrainedShuffleStreamCount\":0,\"Exprs\":[\"mysql.tidb_ddl_job.job_meta\",\"mysql.tidb_ddl_job.processing\"],\"CalculateNoDelay\":false,\"AvoidColumnEvaluator\":false},\"Text\":\"select job_meta, processing from mysql.tidb_ddl_job where job_id in (select min(job_id) from mysql.tidb_ddl_job group by schema_ids, table_ids, processing) and  reorg  order by processing desc, job_id\",\"StmtNode\":{\"SQLBigResult\":false,\"SQLBufferResult\":false,\"SQLCache\":true,\"SQLSmallResult\":false,\"CalcFoundRows\":false,\"StraightJoin\":false,\"Priority\":0,\"ExplicitAll\":false,\"Distinct\":false,\"From\":{\"TableRefs\":{\"Left\":{\"Source\":{\"Schema\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"Name\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"DBInfo\":{\"id\":1,\"db_name\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"charset\":\"utf8mb4\",\"collate\":\"utf8mb4_bin\",\"state\":5,\"policy_ref_info\":null},\"TableInfo\":{\"id\":281474976710654,\"name\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"charset\":\"utf8mb4\",\"collate\":\"utf8mb4_bin\",\"cols\":[{\"id\":1,\"name\":{\"O\":\"job_id\",\"L\":\"job_id\"},\"offset\":0,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":8,\"Flag\":4099,\"Flen\":20,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":2,\"name\":{\"O\":\"reorg\",\"L\":\"reorg\"},\"offset\":1,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":3,\"name\":{\"O\":\"schema_ids\",\"L\":\"schema_ids\"},\"offset\":2,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":250,\"Flag\":0,\"Flen\":16777215,\"Decimal\":0,\"Charset\":\"utf8mb4\",\"Collate\":\"utf8mb4_bin\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":4,\"name\":{\"O\":\"table_ids\",\"L\":\"table_ids\"},\"offset\":3,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":250,\"Flag\":0,\"Flen\":16777215,\"Decimal\":0,\"Charset\":\"utf8mb4\",\"Collate\":\"utf8mb4_bin\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":5,\"name\":{\"O\":\"job_meta\",\"L\":\"job_meta\"},\"offset\":4,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":251,\"Flag\":128,\"Flen\":4294967295,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":6,\"name\":{\"O\":\"type\",\"L\":\"type\"},\"offset\":5,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":7,\"name\":{\"O\":\"processing\",\"L\":\"processing\"},\"offset\":6,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2}],\"index_info\":null,\"constraint_info\":null,\"fk_info\":null,\"state\":5,\"pk_is_handle\":true,\"is_common_handle\":false,\"common_handle_version\":0,\"comment\":\"\",\"auto_inc_id\":0,\"auto_id_cache\":0,\"auto_rand_id\":0,\"max_col_id\":7,\"max_idx_id\":0,\"max_fk_id\":0,\"max_cst_id\":0,\"update_timestamp\":437971042297708545,\"ShardRowIDBits\":0,\"max_shard_row_id_bits\":0,\"auto_random_bits\":0,\"auto_random_range_bits\":0,\"pre_split_regions\":0,\"partition\":null,\"compression\":\"\",\"view\":null,\"sequence\":null,\"Lock\":null,\"version\":5,\"tiflash_replica\":null,\"is_columnar\":false,\"temp_table_type\":0,\"cache_table_status\":0,\"policy_ref_info\":null,\"stats_options\":null,\"exchange_partition_info\":null,\"ttl_info\":null},\"IndexHints\":[],\"PartitionNames\":[],\"TableSample\":null,\"AsOf\":null},\"AsName\":{\"O\":\"\",\"L\":\"\"}},\"Right\":null,\"Tp\":0,\"On\":null,\"Using\":null,\"NaturalJoin\":false,\"StraightJoin\":false,\"ExplicitParens\":false}},\"Where\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Op\":1,\"L\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"job_id\",\"L\":\"job_id\"}},\"Refer\":null},\"List\":null,\"Not\":false,\"Sel\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Query\":{\"SQLBigResult\":false,\"SQLBufferResult\":false,\"SQLCache\":true,\"SQLSmallResult\":false,\"CalcFoundRows\":false,\"StraightJoin\":false,\"Priority\":0,\"ExplicitAll\":false,\"Distinct\":false,\"From\":{\"TableRefs\":{\"Left\":{\"Source\":{\"Schema\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"Name\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"DBInfo\":{\"id\":1,\"db_name\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"charset\":\"utf8mb4\",\"collate\":\"utf8mb4_bin\",\"state\":5,\"policy_ref_info\":null},\"TableInfo\":{\"id\":281474976710654,\"name\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"charset\":\"utf8mb4\",\"collate\":\"utf8mb4_bin\",\"cols\":[{\"id\":1,\"name\":{\"O\":\"job_id\",\"L\":\"job_id\"},\"offset\":0,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":8,\"Flag\":4099,\"Flen\":20,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":2,\"name\":{\"O\":\"reorg\",\"L\":\"reorg\"},\"offset\":1,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":3,\"name\":{\"O\":\"schema_ids\",\"L\":\"schema_ids\"},\"offset\":2,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":250,\"Flag\":0,\"Flen\":16777215,\"Decimal\":0,\"Charset\":\"utf8mb4\",\"Collate\":\"utf8mb4_bin\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":4,\"name\":{\"O\":\"table_ids\",\"L\":\"table_ids\"},\"offset\":3,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":250,\"Flag\":0,\"Flen\":16777215,\"Decimal\":0,\"Charset\":\"utf8mb4\",\"Collate\":\"utf8mb4_bin\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":5,\"name\":{\"O\":\"job_meta\",\"L\":\"job_meta\"},\"offset\":4,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":251,\"Flag\":128,\"Flen\":4294967295,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":6,\"name\":{\"O\":\"type\",\"L\":\"type\"},\"offset\":5,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2},{\"id\":7,\"name\":{\"O\":\"processing\",\"L\":\"processing\"},\"offset\":6,\"origin_default\":null,\"origin_default_bit\":null,\"default\":null,\"default_bit\":null,\"default_is_expr\":false,\"generated_expr_string\":\"\",\"generated_stored\":false,\"dependences\":null,\"type\":{\"Tp\":3,\"Flag\":0,\"Flen\":11,\"Decimal\":0,\"Charset\":\"binary\",\"Collate\":\"binary\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"state\":5,\"comment\":\"\",\"hidden\":false,\"change_state_info\":null,\"version\":2}],\"index_info\":null,\"constraint_info\":null,\"fk_info\":null,\"state\":5,\"pk_is_handle\":true,\"is_common_handle\":false,\"common_handle_version\":0,\"comment\":\"\",\"auto_inc_id\":0,\"auto_id_cache\":0,\"auto_rand_id\":0,\"max_col_id\":7,\"max_idx_id\":0,\"max_fk_id\":0,\"max_cst_id\":0,\"update_timestamp\":437971042297708545,\"ShardRowIDBits\":0,\"max_shard_row_id_bits\":0,\"auto_random_bits\":0,\"auto_random_range_bits\":0,\"pre_split_regions\":0,\"partition\":null,\"compression\":\"\",\"view\":null,\"sequence\":null,\"Lock\":null,\"version\":5,\"tiflash_replica\":null,\"is_columnar\":false,\"temp_table_type\":0,\"cache_table_status\":0,\"policy_ref_info\":null,\"stats_options\":null,\"exchange_partition_info\":null,\"ttl_info\":null},\"IndexHints\":[],\"PartitionNames\":[],\"TableSample\":null,\"AsOf\":null},\"AsName\":{\"O\":\"\",\"L\":\"\"}},\"Right\":null,\"Tp\":0,\"On\":null,\"Using\":null,\"NaturalJoin\":false,\"StraightJoin\":false,\"ExplicitParens\":false}},\"Where\":null,\"Fields\":{\"Fields\":[{\"Offset\":76,\"WildCard\":null,\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"F\":\"min\",\"Args\":[{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"job_id\",\"L\":\"job_id\"}},\"Refer\":null}],\"Distinct\":false,\"Order\":null},\"AsName\":{\"O\":\"\",\"L\":\"\"},\"Auxiliary\":false,\"AuxiliaryColInAgg\":false,\"AuxiliaryColInOrderBy\":false}]},\"GroupBy\":{\"Items\":[{\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"schema_ids\",\"L\":\"schema_ids\"}},\"Refer\":null},\"Desc\":false,\"NullOrder\":true},{\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"table_ids\",\"L\":\"table_ids\"}},\"Refer\":null},\"Desc\":false,\"NullOrder\":true},{\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"processing\",\"L\":\"processing\"}},\"Refer\":null},\"Desc\":false,\"NullOrder\":true}]},\"Having\":null,\"WindowSpecs\":null,\"OrderBy\":null,\"Limit\":null,\"LockInfo\":null,\"TableHints\":null,\"IsInBraces\":false,\"WithBeforeBraces\":false,\"QueryBlockOffset\":2,\"SelectIntoOpt\":null,\"AfterSetOperator\":null,\"Kind\":0,\"Lists\":null,\"With\":null,\"AsViewSchema\":false},\"Evaluated\":false,\"Correlated\":false,\"MultiRows\":true,\"Exists\":false}},\"R\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"reorg\",\"L\":\"reorg\"}},\"Refer\":null}},\"Fields\":{\"Fields\":[{\"Offset\":7,\"WildCard\":null,\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"job_meta\",\"L\":\"job_meta\"}},\"Refer\":null},\"AsName\":{\"O\":\"\",\"L\":\"\"},\"Auxiliary\":false,\"AuxiliaryColInAgg\":false,\"AuxiliaryColInOrderBy\":false},{\"Offset\":17,\"WildCard\":null,\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"processing\",\"L\":\"processing\"}},\"Refer\":null},\"AsName\":{\"O\":\"\",\"L\":\"\"},\"Auxiliary\":false,\"AuxiliaryColInAgg\":false,\"AuxiliaryColInOrderBy\":false}]},\"GroupBy\":null,\"Having\":null,\"WindowSpecs\":null,\"OrderBy\":{\"Items\":[{\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"processing\",\"L\":\"processing\"}},\"Refer\":null},\"Desc\":true,\"NullOrder\":false},{\"Expr\":{\"Type\":{\"Tp\":0,\"Flag\":0,\"Flen\":0,\"Decimal\":0,\"Charset\":\"\",\"Collate\":\"\",\"Elems\":null,\"ElemsIsBinaryLit\":null},\"Name\":{\"Schema\":{\"O\":\"\",\"L\":\"\"},\"Table\":{\"O\":\"\",\"L\":\"\"},\"Name\":{\"O\":\"job_id\",\"L\":\"job_id\"}},\"Refer\":null},\"Desc\":false,\"NullOrder\":true}],\"ForUnion\":false},\"Limit\":null,\"LockInfo\":null,\"TableHints\":null,\"IsInBraces\":false,\"WithBeforeBraces\":false,\"QueryBlockOffset\":1,\"SelectIntoOpt\":null,\"AfterSetOperator\":null,\"Kind\":0,\"Lists\":null,\"With\":null,\"AsViewSchema\":false},\"Ctx\":{},\"LowerPriority\":false,\"OutputNames\":[{\"OrigTblName\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"OrigColName\":{\"O\":\"job_meta\",\"L\":\"job_meta\"},\"DBName\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"TblName\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"ColName\":{\"O\":\"job_meta\",\"L\":\"job_meta\"},\"Hidden\":false,\"NotExplicitUsable\":false,\"Redundant\":false},{\"OrigTblName\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"OrigColName\":{\"O\":\"processing\",\"L\":\"processing\"},\"DBName\":{\"O\":\"mysql\",\"L\":\"mysql\"},\"TblName\":{\"O\":\"tidb_ddl_job\",\"L\":\"tidb_ddl_job\"},\"ColName\":{\"O\":\"processing\",\"L\":\"processing\"},\"Hidden\":false,\"NotExplicitUsable\":false,\"Redundant\":false}],\"PsStmt\":null,\"Ti\":{\"UseNonRecursive\":false,\"UseRecursive\":false,\"UseMultiSchemaChange\":false,\"UesExchangePartition\":false,\"UseFlashbackToCluster\":false,\"PartitionTelemetry\":null,\"AccountLockTelemetry\":null,\"UseIndexMerge\":false}}"]
```

#### Another Reading Session

For each client connection TiDB server calls function (\*clientConn).handleQuery, which calls function Parse inherited from Session interface. Parse returns a set/array/slice of statements. Each statement elements is passed as argument to function (\*clientConn).handleStmt. handleStmt calls (\*TiDBContext).ExecuteStmt (\*session).ExecuteStmt which returns the type sqlexec.RecordSet. It calls (*Compiler).Compile which receives the AST and returns *ExecStmt. ExecStmt is a struct that implements sqlexec.Statement.

Statement and RecordSet interfaces are defined in tidb/util/sqlexec/restricted_sql_executor.go.

type Statement interface {
	// OriginText gets the origin SQL text.
	OriginText() string

	// GetTextToLog gets the desensitization SQL text for logging.
	GetTextToLog() string

	// Exec executes SQL and gets a Recordset.
	Exec(ctx context.Context) (RecordSet, error)

	// IsPrepared returns whether this statement is prepared statement.
	IsPrepared() bool

	// IsReadOnly returns if the statement is read only. For example: SelectStmt without lock.
	IsReadOnly(vars *variable.SessionVars) bool

	// RebuildPlan rebuilds the plan of the statement.
	RebuildPlan(ctx context.Context) (schemaVersion int64, err error)

	// GetStmtNode returns the stmtNode inside Statement
	GetStmtNode() ast.StmtNode
}

// RecordSet is an abstract result set interface to help get data from Plan.
type RecordSet interface {
	// Fields gets result fields.
	Fields() []*ast.ResultField

	// Next reads records into chunk.
	Next(ctx context.Context, req *chunk.Chunk) error

	// NewChunk create a chunk, if allocator is nil, the default one is used.
	NewChunk(chunk.Allocator) *chunk.Chunk

	// Close closes the underlying iterator, call Next after Close will
	// restart the iteration.
	Close() error
}



planner.Optimize

plannercore/preprocess.Preprocess

executor/adapter.(*ExecStmt).IsReadOnly

executor.(*Compiler).Compile

server.(*session).ExecuteStmt

server.(*TiDBContext).ExecuteStmt

server.(*clientConn).handleStmt

server.(*clientConn).handleQuery

server.(*clientConn).dispatch

server.(*clientConn).Run

server.(*Server).onConn

server.(*Server).startNetworkListener

planner/core
tidb/executor/compiler.(*Compiler).Compile

#### No title

[About the TiDB Source Code](https://www.pingcap.com/blog/about-the-tidb-source-code/)

[TiDB Development Guide: ](https://pingcap.github.io/tidb-dev-guide/understand-tidb/introduction.html)

Instead of looking which line function is in the code you can add breakpoints using just functions. See the excerpt of file below you can load during debugging session with the command `source batch.dlv`.

```
break server.(*Server).startNetworkListener
break server.(*clientConn).handleQuery
break executor.(*Compiler).Compile
```

You can also use the command `transcript output.dlv` to save the output of debugging session to the file output.dlv

The command `print valname` can be used to print variables values when debugger stop at breapoints.

There is also the `watch -rw valname` that stops whennever variable is read or written. But I couldn't figure out why it doesn't work with variables of types context.Context and ast.*. 

Besides `print`. How to show variables of context.Context and ast.* types?

#### In the code

executor/compiler.Compile() call at session/session.go:(*session).ExecuteStmt

/home/tidb/session/session.go:2168

planner/planner.Optimize() call at

/home/tidb/executor/compiler.go:116

executor/adapter.ExecStmt.Exec definition at

/home/tidb/executor/adapter.go:426

#### Call Stack

executor.(*Compiler).Compile

server.(*session).ExecuteStmt

server.(*TiDBContext).ExecuteStmt

server.(*clientConn).handleStmt

server.(*clientConn).handleQuery

server.(*clientConn).dispatch

server.(*clientConn).Run

server.(*Server).onConn

server.(*Server).startNetworkListener



#### Interesting breakpoints:

func (s *Server) startNetworkListener(listener net.Listener, isUnixSocket bool, errChan chan error)


func (s *session) Parse(ctx context.Context, sql string) ([]ast.StmtNode, error)
/home/tidb/session/session.go:1712

func (s *SessionVars) GetParseParams() []parser.ParseParam
/home/tidb/sessionctx/variable/session.go:1862

func (s *session) ParseSQL(ctx context.Context, sql string, params ...parser.ParseParam) ([]ast.StmtNode, []error, error)
/home/tidb/session/session.go:1545
sql: "BEGIN PESSIMISTIC" ?

func Optimize(ctx context.Context, sctx sessionctx.Context, node ast.Node, is infoschema.InfoSchema) (core.Plan, types.NameSlice, error)
/home/tidb/planner/optimize.go:107


## Using `pprof` for profiling

For any database system, performance is always important. If you want to know where the performance bottleneck is, you can use a powerful Go profiling tool called `pprof`.

### Gather runtime profiling information through HTTP end points

Usually, when TiDB server is running, it exposes a profiling end point through HTTP at `http://127.0.0.1:10080/debug/pprof/profile`. You can get the profile result by running the following commands:

```bash
curl -G "127.0.0.1:10080/debug/pprof/profile?seconds=45" > profile.profile
go tool pprof -http 127.0.0.1:4001 profile.profile
```

The commands capture the profiling information for 45 seconds, and then provide a web view of the profiling result at `127.0.0.1:4001`. This view contains a [flame graph](http://www.brendangregg.com/flamegraphs.html) of the execution and more views that can help you diagnose the performance bottleneck.

You can also gather other runtime information through this end point. For example:

* Goroutine:

```bash
curl -G "127.0.0.1:10080/debug/pprof/goroutine" > goroutine.profile
```

* Trace:

```bash
curl -G "127.0.0.1:10080/debug/pprof/trace?seconds=3" > trace.profile
go tool trace -http 127.0.0.1:4001 trace.profile
```

* Heap:

```bash
curl -G "127.0.0.1:10080/debug/pprof/heap" > heap.profile
go tool pprof -http 127.0.0.1:4001 heap.profile
```

To learn how the runtime information is analyzed, see Go's [diagnostics document](https://golang.org/doc/diagnostics).

### Profiling during benchmarking

When you are proposing a performance-related feature for TiDB, we recommend that you also include a benchmark result as proof of the performance gain or to show that your code won't introduce any performance regression. In this case, you need to write your own benchmark test like in `executor/benchmark.go`.

For example, if you want to benchmark the window functions, because `BenchmarkWindow` are already in the benchmark tests, you can run the following commands to get the benchmark result:

```bash
cd executor
go test -bench BenchmarkWindow -run BenchmarkWindow -benchmem
```

If you find any performance regression, and you want to know the cause of it, you could use a command like the following:

```bash
go test -bench BenchmarkWindow -run BenchmarkWindow -benchmem -memprofile memprofile.out -cpuprofile profile.out
```

Then, you can use the steps described above to generate and analyze the profiling information.
