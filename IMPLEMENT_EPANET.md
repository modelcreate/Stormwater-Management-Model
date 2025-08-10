### EPANET Implementation Guide: Named Variables & Arithmetic Expressions (in [RULES])

This plan details exact EPANET code changes to support SWMM 5.2-style Named Variables and Arithmetic Expressions, defined and used entirely within the `[RULES]` section. No new public APIs will be added; the engine will parse and use them internally at run time. Validate each step against the checklists at the bottom.

References: see `RESEARCH_SWMM.md` for SWMM implementation specifics.

### Scope and Syntax (EPANET)
- New `[RULES]` lines (order-agnostic, prior to first use):
  - `VARIABLE name = OBJECT id ATTRIBUTE`
  - `VARIABLE name = SYSTEM ATTRIBUTE`
  - `EXPRESSION name = <math expression over named variables>`
- Clause extensions:
  - `IF/AND/OR <variableName> <relop> <value|variable|OBJECT id ATTRIBUTE>`
  - `IF/AND/OR <expressionName> <relop> <value|variable|OBJECT id ATTRIBUTE>`
- Supported functions/operators in expressions: same set as SWMM (`abs`, `sgn`, `step`, trig incl. hyperbolic and inverses, `log`, `log10`, `exp`, operators `+ - * / ^` and parentheses).
- Allowed OBJECT/ATTRIBUTE in variables match EPANET’s rule variables (`NODE` or `LINK` and `SYSTEM` with `TIME`/`CLOCKTIME`, etc.).

### File-by-File Changes

1) `epanet/src/text.h`
- Add keywords:
  - `#define w_VARIABLE   "VARIABLE"`
  - `#define w_EXPRESSION "EXPRESSION"`

2) `epanet/src/rules.c`
- Extend Ruleword enum and table to recognize new tokens:
  - enum: add `r_VARIABLE`, `r_EXPRESSION` before `r_ERROR`
  - array: insert `w_VARIABLE`, `w_EXPRESSION`
- Storage and counts (Rules wrapper; see types.h below) will be available via `pr->rules`.
- Counting pass: modify `addrule(Parser*, char*)` to increment counts:
  - if `match(tok, w_RULE)`: `parser->MaxRules++`
  - else if `match(tok, w_VARIABLE)`: `pr->rules.VarCount++`
  - else if `match(tok, w_EXPRESSION)`: `pr->rules.ExprCount++`
- Allocation: in `allocrules(Project*)`, after allocating `net->Rule`, allocate `pr->rules.NamedVars` and `pr->rules.Expressions` using `VarCount` and `ExprCount` (set `CurrentVar = CurrentExpr = -1`).
- Freeing: in `freerules(Project*)`, free compiled expressions and arrays (see types.h changes) in addition to existing rule cleanup.
- Parsing (`ruledata(Project*)`): add cases for `r_VARIABLE` and `r_EXPRESSION` that operate regardless of `RuleState` (but before `RULE` body begins is recommended):
  - `r_VARIABLE`: parse formats
    - `VARIABLE name = SYSTEM <Varword>`
    - `VARIABLE name = <OBJECT> <id> <Varword>`
    - Reject if `name` equals any Varword or conflicts with reserved keywords.
    - Resolve to object/index/variable triple with existing helpers: `findmatch(Object,...)`, `findnode/findlink`, `findmatch(Varword,...)`.
    - Store in `NamedVars[++CurrentVar] = {name, object, index, variable}`.
  - `r_EXPRESSION`: assemble expression string from `Tok[3..Ntokens-1]`; compile via `mathexpr_create(exprString, getVariableIndex)` and store in `Expressions[++CurrentExpr] = {name, expression}`; on failure set rule parser error (`Errcode=201/202` mapped to new math expr error code if desired) and `RuleState=r_ERROR`.
- Premise parsing (`newpremise(Project*, int)`): expand to accept LHS as expression or named variable:
  - Before current object parsing, check if `Tok[1]` matches an expression name; if yes, set `p->exprIndex = exprIdx` and skip object parsing; parse relop at `Tok[2]`, then parse RHS as value or variable (named or object triple). Set `p->rhsIsVar=1` and populate RHS triple if variable, else assign `p->value`.
  - Else check if `Tok[1]` matches a named variable; if yes, set `p->object/index/variable` from that alias; then continue current relop and RHS logic; if RHS is a variable (named or object triple), set RHS fields; else set `p->value`.
  - Else fall back to existing formats (SYSTEM or explicit OBJECT id VAR).
  - Validation: when both sides are variables and LHS is not an expression, if different variable kinds (e.g., `STATUS` vs `FLOW`), return 201 or optionally log a warning (consistent with SWMM’s WARN11 semantics).
- Evaluation:
  - `checkpremise(Project*, Spremise*)` remains dispatcher, but when `p->exprIndex >= 0` handle in `checkvalue` branch.
  - `checkvalue(Project*, Spremise*)` modifications:
    - Compute LHS value:
      - If `exprIndex >= 0`, evaluate via `mathexpr_eval(Expressions[exprIndex].expression, getNamedVariableValue)`.
      - Else compute as today based on `p->object/index/variable` (existing switch).
    - Compute RHS value:
      - If `p->rhsIsVar`, compute value using the same logic as for LHS (object/index/variable triple), or if RHS references a named variable, resolve it pre-parsed to object/index/variable.
      - Else use `p->value`.
    - Compare using existing tolerance logic on numeric values; time/clocktime stays in `checktime()`.

3) `epanet/src/types.h`
- Extend `Spremise` to support expressions and RHS variables:
  - Add fields (initialized to defaults where constructed):
    - `int exprIndex;      // -1 if not an expression LHS`
    - `int rhsIsVar;       // 1 if RHS is a variable`
    - `int rhsObject;      // r_NODE/r_LINK/r_SYSTEM`
    - `int rhsIndex;       // object index`
    - `int rhsVariable;    // Varwords enum`
- Extend `Rules` wrapper to store named variables and expressions:
  - Counts and cursors: `int VarCount, ExprCount, CurrentVar, CurrentExpr;`
  - Name length: define `#define MAXVARNAME 32` in `rules.c` or a shared header and use here accordingly.
  - Named variables array:
    - `typedef struct { char name[MAXVARNAME+1]; int object; int index; int variable; } NamedVar;`
    - `NamedVar *NamedVars;`
  - Expressions array:
    - Forward-declare `typedef struct ExprNode MathExpr;` (from `mathexpr.h`)
    - `typedef struct { char name[MAXVARNAME+1]; MathExpr* expr; } NamedExpr;`
    - `NamedExpr *Expressions;`

4) `epanet/src/input2.c`
- First pass counting already routes `[RULES]` tokens to `addrule(parser, tok)`. With the changes above, `addrule` will handle `VARIABLE` and `EXPRESSION` counts alongside `RULE`.

5) `epanet/src/project.c` (only if needed)
- Ensure lifecycle calls to `initrules`, `allocrules`, and `freerules` happen as they are today (they do). No public API changes.

6) New Module: `epanet/src/mathexpr.c` and `epanet/src/mathexpr.h`
- Add SWMM’s `mathexpr.c/h` (unmodified API):
  - `MathExpr* mathexpr_create(char*, int (*getVar)(char*))`
  - `double mathexpr_eval(MathExpr*, double (*getVal)(int))`
  - `void mathexpr_delete(MathExpr*)`
- In `rules.c`, implement two callbacks:
  - `static int getVariableIndex(char* name)` that scans `pr->rules.NamedVars` for a case-insensitive name match and returns index or -1.
  - `static double getNamedVariableValue(int idx)` that converts `NamedVars[idx]` to a current numeric value using the same logic as `checkvalue` (reuse helper to compute a (object,index,variable) value with unit conversion and tolerances).

### Parsing Details (token positions)
- VARIABLE lines:
  - Tokens: `[0]=VARIABLE`, `[1]=name`, `[2]==`, then either `[3]=SYSTEM [4]=Var` or `[3]=OBJECT [4]=id [5]=Var`.
  - Reject if fewer than 5 tokens or bad keyword/operator.
  - Name cannot match any `Varword` or reserved rule keywords.
- EXPRESSION lines:
  - Tokens: `[0]=EXPRESSION`, `[1]=name`, `[2]==`, `[3..]` concatenated with single spaces to form the formula string.
- Premises with names/expressions:
  - Expression LHS: `[1]=ExprName`, `[2]=relop`, RHS begins at `[3]`.
  - NamedVar LHS: `[1]=VarName`, `[2]=<relop>`, RHS at `[3]`.
  - For RHS variable, accept either another named var or `OBJECT id Var` triple; else parse number/status as today. For LINK/NODE status/setting, keep existing status parsing when user uses `IS OPEN/CLOSED/ACTIVE`.

### Evaluation Rules
- Time/clocktime premises remain evaluated by `checktime()`.
- Numeric premises use `checkvalue()`; when an expression is used, units are those of each variable’s native outputs in EPANET, combined algebraically. It’s the user’s responsibility to maintain consistent units.
- Status comparisons remain via `checkstatus()`.
- When comparing variable-vs-variable, use the same 0.001 tolerance; if the compared variables differ in dimension (e.g., `FLOW` vs `PRESSURE`), mark rule parse error or simply return false and log a warning (align to SWMM’s WARN11 approach—EPANET can log a parser message and continue).

### Writing Back to INP (optional, later)
- `writerule()` currently writes only the IF/THEN/ELSE/Priority clauses. If needed later, prepend the set of `VARIABLE` and `EXPRESSION` lines used by any rule to the `[RULES]` section, preserving order of definition.

### Error Handling
- Use existing rule parser error codes:
  - `201` unrecognized/misplaced token or bad attribute
  - `202` bad number/time literal
  - `203` bad node id; `204` bad link id
  - For math expression compile error, map to `201` or introduce a dedicated code if desirable.
- Name conflicts: if `VARIABLE name` matches an existing Varword or rule keyword, treat as `201`.

### Internal-Only Principle
- No toolkit API changes. Engine parses and evaluates named variables and expressions only from `[RULES]` in input files and in `EN_addrule()` text ingestion (which routes text into `ruledata()` already).

### Testing Plan
- Unit tests (parser):
  - VARIABLE lines (SYSTEM TIME/CLOCKTIME; NODE/LINK variables)
  - EXPRESSION lines (simple arithmetic; trig/log; nested parentheses; power)
  - Premises using expression vs value; named var vs value; named var vs named var; named var vs object triple
  - Invalid names, missing tokens, bad function names, bad object/variable names
- Integration tests:
  - Small networks with `[RULES]` using variables and expressions to toggle link status/setting
  - Check evaluation cadence is unaffected (uses existing `Rulestep`)
  - Regression on existing rule models to ensure backward compatibility

### Checklists

- Phase A — Named Variables
  - [ ] Add `w_VARIABLE`, Ruleword enum/table entries
  - [ ] Count in `addrule` and allocate arrays in `allocrules`
  - [ ] Extend `Spremise` with RHS variable fields (can be added now for both phases)
  - [ ] Parse `VARIABLE` lines into `NamedVars`
  - [ ] Resolve named vars in `newpremise` LHS/RHS
  - [ ] Compute named var values in evaluation path
  - [ ] Unit/integration tests green

- Phase B — Arithmetic Expressions
  - [ ] Add `w_EXPRESSION`, Ruleword enum/table entries
  - [ ] Count in `addrule` and allocate `Expressions`
  - [ ] Add `exprIndex` to `Spremise`; parse expression LHS
  - [ ] Integrate `mathexpr.c/h` and implement callbacks
  - [ ] Evaluate expressions via `mathexpr_eval` in `checkvalue`
  - [ ] Variable-vs-variable comparisons in RHS supported
  - [ ] Tests for functions/operators and mixed cases green

- Phase C — Polishing
  - [ ] Free compiled expressions in `freerules`
  - [ ] Optional: writer enhancements for VARIABLE/EXPRESSION
  - [ ] Backward compat verified on existing test suite

Notes:
- Keep differences from SWMM minimal for maintainability; we mirror its architecture (named variable aliasing, expression parser callbacks) while aligning with EPANET’s rule and evaluation code.
- Storage is attached to `Rules` to localize lifetime and avoid public API exposure.