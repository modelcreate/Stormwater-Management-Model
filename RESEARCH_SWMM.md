### SWMM 5.2 Implementation Research: Named Variables & Arithmetic Expressions

This document inventories the exact files, structs, functions, and control flow in SWMM 5.2 that implement Named Variables and Arithmetic Expressions for control rules, including how they are parsed from INP, stored, and evaluated during simulation.

### Key Files
- `swmm/src/solver/controls.c`: rule parsing, named variables, expressions, evaluation
- `swmm/src/solver/input.c`: INP parsing, rule section handling, counts
- `swmm/src/solver/mathexpr.c` and `mathexpr.h`: expression tokenizer, parser, evaluator
- `swmm/src/solver/keywords.c` and `text.h`: control-specific keywords
- `swmm/src/solver/routing.c` and `project.c`: simulation integration points

### Keywords & Tokens
- Control keywords: `w_RULE`, `w_IF`, `w_AND`, `w_OR`, `w_THEN`, `w_ELSE`, `w_PRIORITY`
- New tokens in 5.2: `w_VARIABLE`, `w_EXPRESSION` (see `text.h` and `keywords.c`)

```1:20:swmm/src/solver/text.h
#define  w_RULE              "RULE"
#define  w_IF                "IF"
#define  w_AND               "AND"
#define  w_OR                "OR"
#define  w_ELSE              "ELSE"
#define  w_PRIORITY          "PRIORITY"
#define  w_VARIABLE          "VARIABLE"
#define  w_EXPRESSION        "EXPRESSION"
```

```113:120:swmm/src/solver/keywords.c
char* RuleKeyWords[]       = { w_RULE, w_IF, w_AND, w_OR, w_THEN, w_ELSE, 
                               w_PRIORITY, NULL};
```

### Data Structures (controls.c)
- `TVariable`: identifies an attribute of an object (object type, index, attribute)
- `TNamedVariable`: maps a `name` to a `TVariable`
- `TExpression`: stores an expression `name` and compiled `MathExpr*`
- `TPremise`: one clause; holds either an expression index on LHS or an LHS `TVariable`, a relation, and an RHS variable or numeric value
- `TAction`, `TActionList`, `TRule`: actions and rule definitions

```95:116:swmm/src/solver/controls.c
struct TVariable { int object; int index; int attribute; };
struct TNamedVariable { struct TVariable variable; char name[MAXVARNAME+1]; };
struct TExpression { MathExpr* expression; char name[MAXVARNAME+1]; };
struct TPremise { int type; int exprIndex; struct TVariable lhsVar; struct TVariable rhsVar; int relation; double value; struct TPremise *next; };
```

Shared globals for rule system:
- `Rules`, `ActionList`, `RuleCount`
- `NamedVariable` (array), `Expression` (array), their counts and current indices

```164:179:swmm/src/solver/controls.c
int     VariableCount; int ExpressionCount; int CurrentVariable; int CurrentExpression;
struct  TNamedVariable* NamedVariable; struct  TExpression* Expression;
```

### INP Reading – Counting and Parsing
1) First pass counts variables/expressions and rules

```88:96:swmm/src/solver/input.c
for (i = 0; i < MAX_OBJ_TYPES; i++) Nobjects[i] = 0; ...
controls_init();
...
case s_CONTROL:
  if ( match(id, w_RULE) ) Nobjects[CONTROL]++;
  else controls_addToCount(id);
```

- `controls_addToCount` increments `VariableCount` or `ExpressionCount` if the line starts with the respective keyword.

```243:253:swmm/src/solver/controls.c
void controls_addToCount(char* s) { if (match(s, w_VARIABLE)) VariableCount++; else if (match(s, w_EXPRESSION)) ExpressionCount++; }
```

2) Allocation during project create

```1051:1056:swmm/src/solver/project.c
// --- create control rules
ErrorCode = controls_create(Nobjects[CONTROL]);
```

```283:297:swmm/src/solver/controls.c
if (VariableCount > 0) NamedVariable = calloc(VariableCount, sizeof(TNamedVariable));
if (ExpressionCount > 0) Expression = calloc(ExpressionCount, sizeof(TExpression));
```

3) Second pass parses lines in `[CONTROLS]`

```648:656:swmm/src/solver/input.c
if (match(tok[0], w_VARIABLE)) return controls_addVariable(tok, ntoks);
if (match(tok[0], w_EXPRESSION)) return controls_addExpression(tok, ntoks);
// otherwise, a RULE clause
keyword = findmatch(tok[0], RuleKeyWords);
return controls_addRuleClause(index, keyword, Tok, Ntokens);
```

- `controls_addVariable` enforces uniqueness vs attribute names and parses `VARIABLE name = <object ...>` or `SIMULATION` variants.

```340:353:swmm/src/solver/controls.c
CurrentVariable++;
if (findExactMatch(tok[1], AttribWords) >= 0) return error_setInpError(ERR_KEYWORD, tok[1]);
... getPremiseVariable(...,&v1);
NamedVariable[k].variable = v1; sstrncpy(NamedVariable[k].name, tok[1], MAXVARNAME);
```

- `controls_addExpression` composes the formula string and compiles it with `mathexpr_create`, providing a callback `getVariableIndex` to resolve named-variable identifiers.

```371:389:swmm/src/solver/controls.c
Expression[k].expression = NULL; sstrncpy(Expression[k].name, tok[1], MAXVARNAME);
// join tokens[3..]
expr = mathexpr_create(s, getVariableIndex);
Expression[k].expression = expr;
```

### Parsing Variables in Clauses
- For a premise, LHS can be either an expression name or a named variable; otherwise it’s an object/id/attribute triple parsed by `getPremiseVariable`.

```580:599:swmm/src/solver/controls.c
exprIndex = getExpressionIndex(tok[1]);
if (exprIndex < 0) { varIndex = getVariableIndex(tok[n]); if (varIndex >= 0) v1 = NamedVariable[varIndex].variable; else err = getPremiseVariable(...,&v1); }
```

- RHS can be a named variable, another object/id/attribute, or a literal value parsed by `getPremiseValue`.

```613:637:swmm/src/solver/controls.c
varIndex = getVariableIndex(tok[n]); if (varIndex >= 0) v2 = NamedVariable[varIndex].variable; else { obj = findmatch(tok[n], ObjectWords); if (obj >= 0) getPremiseVariable(...,&v2); else getPremiseValue(tok[n], v1.attribute, &value); }
```

### Expression and Variable Lookup
- `getVariableIndex(name)` scans `NamedVariable[]` for a match
- `getNamedVariableValue(idx)` fetches live value via `getVariableValue(TVariable)`
- `getExpressionIndex(name)` scans `Expression[]`

```399:417:swmm/src/solver/controls.c
for (i=0;i<VariableCount;i++) if (match(varName, NamedVariable[i].name)) return i;
...
return getVariableValue(NamedVariable[varIndex].variable);
```

### Runtime Evaluation in Simulation
- Rules evaluated each routing step when rule-time is reached

```278:283:swmm/src/solver/routing.c
controls_evaluate(currentDate, currentDate - StartDateTime, routingStep / SECperDAY);
```

- Evaluation iterates rules, evaluates premises, and enqueues actions

```509:552:swmm/src/solver/controls.c
while (p) { result = (p->type==r_OR) ? (result || evaluatePremise(p,tStep)) : (result && evaluatePremise(p,tStep)); p=p->next; }
... updateActionValue(...); updateActionList(...);
```

- Premise evaluation: if LHS is an expression, evaluate with `mathexpr_eval`, passing callback to resolve named variable values; otherwise evaluate LHS variable value via `getVariableValue`. RHS is either a variable value or literal. Comparison uses `compareTimes` or `compareValues`.

```1254:1281:swmm/src/solver/controls.c
if (p->exprIndex >= 0) lhsValue = mathexpr_eval(Expression[p->exprIndex].expression, getNamedVariableValue);
else lhsValue = getVariableValue(p->lhsVar);
if (p->value == MISSING) rhsValue = getVariableValue(p->rhsVar); else rhsValue = p->value;
... switch(p->lhsVar.attribute) { case r_TIME... default: return compareValues(...); }
```

- `getVariableValue` supports SIMULATION time/date/day/month, and object attributes for nodes and links, plus rain gage attributes.

```1296:1383:swmm/src/solver/controls.c
case r_TIME: return ElapsedTime; case r_DATE: return CurrentDate; case r_CLOCKTIME: return CurrentTime; ... diverse node/link attributes
```

### Expression Engine (mathexpr)
- Tokenizer supports numbers, variable names (resolved by `getVariableIndex`), functions, and operators; builds an expression tree then a postfix linked list (`MathExpr`)
- Supported functions and operators match the manual; evaluator uses a local stack and callback `getVariableValue(int)` to fetch the live value of named variables

```80:85:swmm/src/solver/mathexpr.c
char *MathFunc[] = {"COS","SIN","TAN","COT","ABS","SGN","SQRT","LOG","EXP","ASIN","ACOS","ATAN","ACOT","SINH","COSH","TANH","COTH","LOG10","STEP", NULL};
```

```516:728:swmm/src/solver/mathexpr.c
double mathexpr_eval(MathExpr *expr, double (*getVariableValue) (int)) { ... switch(node->opcode) { case 3:+, 4:-, 5:*, 6:/, 31:^, 7:number, 8:variable (via callback), 9:negation, 10..28: math functions } }
```

### INP Output/Writing
- SWMM core does not round-trip write INP files from the solver; this is typically handled in UI projects. The core does not include a writer for `[CONTROLS]` back to INP. The features are fully supported for reading and runtime evaluation.

### Error Handling & Validation
- `controls_addVariable` rejects variable names that collide with attribute keywords
- `controls_addExpression` returns `ERR_MATH_EXPR` if the expression fails to compile
- `addPremise` warns if LHS and RHS attributes differ when both are variables (non-expression)

```342:353:swmm/src/solver/controls.c
if (findExactMatch(tok[1], AttribWords) >= 0) return error_setInpError(ERR_KEYWORD, tok[1]);
```

```383:386:swmm/src/solver/controls.c
expr = mathexpr_create(s, getVariableIndex);
if (expr == NULL) return error_setInpError(ERR_MATH_EXPR, "");
```

```626:630:swmm/src/solver/controls.c
if (exprIndex < 0 && v1.attribute != v2.attribute) report_writeWarningMsg(WARN11, Rules[r].ID);
```

### Summary of Control Flow
- Count pass: detect `VARIABLE` and `EXPRESSION` lines, count rules
- Create: allocate arrays for rules, named vars, expressions
- Parse pass: add variables and expressions; build rules with possible expression LHS and variable RHS
- Simulation: at each rule evaluation step, evaluate clause values; expression LHS uses `mathexpr_eval` with named variable callback
- Execute: apply actions with priority handling

### Notes for Porting
- The expression engine is self-contained (`mathexpr.c/h`) with callback hooks for variable name resolution and value retrieval
- Named variables are resolved by name at parse/compile time and evaluated by index at runtime
- Premise clauses allow either expressions or plain variables on LHS; RHS is variable or literal

### Enums and Keywords in `controls.c`
- Enums: `RuleState`, `RuleObject`, `RuleAttrib`, `RuleRelation`, `RuleSetting`

```65:75:swmm/src/solver/controls.c
enum RuleState    {r_RULE, r_IF, r_AND, r_OR, r_THEN, r_ELSE, r_PRIORITY,
                   r_VARIABLE, r_EXPRESSION, r_ERROR};
enum RuleObject   {r_GAGE, r_NODE, r_LINK, r_CONDUIT, r_PUMP, r_ORIFICE,
                   r_WEIR, r_OUTLET, r_SIMULATION};
enum RuleAttrib   {r_DEPTH, r_MAXDEPTH, r_HEAD, r_VOLUME, r_INFLOW,
                   r_FLOW, r_FULLFLOW, r_FULLDEPTH, r_STATUS, r_SETTING,
                   r_LENGTH, r_SLOPE, r_VELOCITY, r_TIMEOPEN, r_TIMECLOSED,
                   r_TIME, r_DATE, r_CLOCKTIME, r_DAYOFYEAR, r_DAY, r_MONTH};
enum RuleRelation {EQ, NE, LT, LE, GT, GE};
enum RuleSetting  {r_CURVE, r_TIMESERIES, r_PID, r_NUMERIC};
```

- Keyword tables: `ObjectWords[]`, `AttribWords[]`, `RelOpWords[]`, `StatusWords[]`, `ConduitWords[]`, `SettingTypeWords[]`

```78:90:swmm/src/solver/controls.c
static char* ObjectWords[] = {"GAGE","NODE","LINK","CONDUIT","PUMP","ORIFICE","WEIR","OUTLET","SIMULATION", NULL};
static char* AttribWords[] = {"DEPTH","MAXDEPTH","HEAD","VOLUME","INFLOW","FLOW","FULLFLOW","FULLDEPTH","STATUS","SETTING","LENGTH","SLOPE","VELOCITY","TIMEOPEN","TIMECLOSED","TIME","DATE","CLOCKTIME","DAYOFYEAR","DAY","MONTH", NULL};
```

### Core Functions Inventory (with roles)
- Lifecycle
  - `controls_init()` initializes globals
  - `controls_addToCount(char* s)` counts VARIABLE/EXPRESSION lines
  - `controls_create(int n)` allocates arrays for rules, named variables, expressions
  - `controls_delete()` frees expressions, variables, rules, and action list

```226:239:swmm/src/solver/controls.c
void controls_init() { Rules = NULL; NamedVariable = NULL; Expression = NULL; RuleCount = 0; VariableCount = 0; ExpressionCount = 0; }
```

- INP Parsing and Rule Building
  - `controls_addVariable(char* tok[], int nToks)` adds named variable
  - `controls_addExpression(char* tok[], int nToks)` compiles and stores expression
  - `controls_addRuleClause(int r, int keyword, char* tok[], int nToks)` dispatches to `addPremise` or `addAction`
  - `addPremise(...)` parses a premise: possible expression LHS or named variable, relation, and RHS variable or value
  - `getPremiseVariable(...)` parses `<OBJECT ID ATTRIBUTE>` or `SIMULATION ATTRIBUTE`
  - `getPremiseValue(...)` parses RHS literal given attribute context (supports time/date/status)
  - `addAction(...)` parses control actions and optional modulated settings (Curve, Time Series, PID)
  - `setActionSetting(...)` handles action setting types

```556:664:swmm/src/solver/controls.c
int addPremise(int r, int type, char* tok[], int nToks) { ... p->exprIndex = exprIndex; p->lhsVar = v1; p->rhsVar = v2; p->relation = relation; p->value = value; }
```

```668:793:swmm/src/solver/controls.c
int getPremiseVariable(char* tok[], int nToks, int* k, struct TVariable* v);
```

```824:885:swmm/src/solver/controls.c
int getPremiseValue(char* token, int attrib, double* value);
```

```889:1024:swmm/src/solver/controls.c
int addAction(int r, char* tok[], int nToks); // validates object type, attribute, and setting
```

```1028:1082:swmm/src/solver/controls.c
int setActionSetting(char* tok[], int nToks, int* curve, int* tseries, int* attrib, double values[]);
```

- Expression/Variable Lookup
  - `getVariableIndex(char* varName)`; `getNamedVariableValue(int varIndex)`
  - `getExpressionIndex(char* exprName)`

```393:417:swmm/src/solver/controls.c
int getVariableIndex(char* varName);
double getNamedVariableValue(int varIndex);
int getExpressionIndex(char* exprName);
```

- Evaluation
  - `controls_evaluate(DateTime currentTime, DateTime elapsedTime, double tStep)` iterates rules, evaluates premises, updates action list, executes actions
  - `evaluatePremise(struct TPremise* p, double tStep)` computes lhs/rhs and compares
  - `getVariableValue(struct TVariable v)` retrieves live values
  - `compareTimes(...)`, `compareValues(...)`
  - Action list helpers: `updateActionValue`, `getPIDSetting`, `updateActionList`, `executeActionList`, `clearActionList`, `deleteActionList`, `deleteRules`

```495:552:swmm/src/solver/controls.c
int controls_evaluate(DateTime currentTime, DateTime elapsedTime, double tStep);
```

```1243:1281:swmm/src/solver/controls.c
int evaluatePremise(struct TPremise* p, double tStep);
```

```1286:1384:swmm/src/solver/controls.c
double getVariableValue(struct TVariable v);
```

### Rule Evaluation Schedule
- Rule evaluation is invoked from the routing step logic when the rule time is reached (uses internal scheduling like `RuleStep`)

```278:283:swmm/src/solver/routing.c
if (RuleStep == 0 || fabs(NewRoutingTime - NewRuleTime) < 1.0)
{  controls_evaluate(currentDate, currentDate - StartDateTime, routingStep / SECperDAY); }
```

- `w_RULE_STEP` option keyword exists to configure rule evaluation frequency

```77:86:swmm/src/solver/text.h
#define  w_RULE_STEP         "RULE_STEP"
```

### Expression Engine Details
- Creation and evaluation API:

```26:33:swmm/src/solver/mathexpr.h
MathExpr* mathexpr_create(char* s, int (*getVar) (char *));
double    mathexpr_eval(MathExpr* expr, double (*getVal) (int));
void      mathexpr_delete(MathExpr* expr);
```

- Operators and functions supported (opcode mapping):

```14:44:swmm/src/solver/mathexpr.c
// 7 number, 8 variable, 31 ^, and functions: cos,sin,tan,cot,abs,sgn,sqrt,log,exp,asin,acos,atan,acot,sinh,cosh,tanh,coth,log10,step
```

### INP Read/Write Summary
- Read: `[CONTROLS]` section processed in two passes; variable/expression lines handled before rule clauses.
- Write: The SWMM solver library does not implement INP writing of control definitions; UI/front-ends handle INP serialization. Only input summaries are written (`inputrpt.c`).

### Cross-References and Warnings
- Mixed-attribute variable-to-variable comparisons (non-expression) generate warning `WARN11` during premise parsing.

```626:630:swmm/src/solver/controls.c
if (exprIndex < 0 && v1.attribute != v2.attribute)
    report_writeWarningMsg(WARN11, Rules[r].ID);
```
