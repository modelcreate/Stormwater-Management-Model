### EPANET Implementation Guide: Named Variables & Arithmetic Expressions

This guide provides a phased plan to integrate SWMM 5.2-style Named Variables and Arithmetic Expressions into EPANET. It maps SWMM components to EPANET integration points and defines testable milestones.

References: see `RESEARCH_SWMM.md` for exact SWMM implementation details.

### EPANET Codebase Integration Points
- Parser and input handling: `epanet/src/epanet/src/input1.c`, `input2.c`, `input3.c` (varies by branch); control rules in `rules.c` if present (verify exact file names in your version)
- Control rule evaluation: often in `rules.c`/`controls.c` or integrated in hydraulics solver loop (`hydraul.c` / `runhyd.c`)
- Keyword tables: `text.h` / `keywords.c` equivalent (confirm EPANET files)

Action: Confirm file names in your EPANET version and update this mapping accordingly.

### Phase A — Named Variables
Goal: Support `VARIABLE name = <OBJECT ID ATTRIBUTE>` and `VARIABLE name = SIMULATION <attribute>` definitions, and allow their use in control rule premises.

1) Keywords & Counting
- Add keywords: `VARIABLE`
- In the first parsing/count pass, count named variables to pre-allocate storage

2) Data Structures
- Add `TVariable { int object; int index; int attribute; }`
- Add `TNamedVariable { TVariable variable; char name[MAXVARNAME+1]; }`
- Add globals: `NamedVariable*`, `VariableCount`, `CurrentVariable`

3) INP Parsing
- In `[CONTROLS]`, before `RULE` clauses, accept `VARIABLE` lines
- Implement `controls_addVariable(tok[], ntoks)` with SWMM behavior:
  - Disallow variable names colliding with attribute keywords
  - Parse `SIMULATION` attribute or `<OBJECT ID ATTRIBUTE>` via a helper `getPremiseVariable`

4) Rule Premises
- Extend premise parser so that the LHS token can be a named variable; else parse as object triple
- On RHS, allow named variable, object triple, or literal value

5) Evaluation
- Extend variable evaluation function to support `SIMULATION` time/date attributes (time since start, date, etc.) and EPANET object attributes relevant to controls

6) Tests
- Unit: parsing `VARIABLE` lines (valid/invalid)
- Integration: control rules using named variables in simple networks

Checklist
- [ ] `VARIABLE` keyword added and counted
- [ ] `TNamedVariable` and storage added
- [ ] `controls_addVariable` implemented
- [ ] Premise parser recognizes named variables
- [ ] Variable evaluation returns correct values
- [ ] Tests pass

### Phase B — Arithmetic Expressions
Goal: Support `EXPRESSION name = <math expression of named variables>` and allow expressions as LHS in premises.

1) Keywords & Counting
- Add keyword: `EXPRESSION`
- Count expressions during the first pass; pre-allocate storage

2) Data Structures
- Add `TExpression { MathExpr* expression; char name[MAXVARNAME+1]; }`
- Add globals: `Expression*`, `ExpressionCount`, `CurrentExpression`

3) Expression Parser Integration
- Port `mathexpr.c/h` from SWMM verbatim or as a new module in EPANET, preserving API:
  - `MathExpr* mathexpr_create(char* s, int (*getVar)(char*))`
  - `double mathexpr_eval(MathExpr* expr, double (*getVal)(int))`
  - `void mathexpr_delete(MathExpr* expr)`
- Implement `getVariableIndex(name)` and `getNamedVariableValue(idx)` for EPANET to connect named variables to expression parser

4) INP Parsing
- Implement `controls_addExpression(tok[], ntoks)` assembling the formula string and compiling with `mathexpr_create`

5) Rule Premises
- Extend premise structure to include `exprIndex` and set it when the LHS token matches a defined expression name

6) Evaluation
- In premise evaluation, if `exprIndex >= 0`, compute LHS = `mathexpr_eval(Expression[exprIndex].expression, getNamedVariableValue)`; otherwise evaluate LHS variable as before

7) Tests
- Unit: expression parsing (operators, precedence, functions)
- Integration: rules with expressions (variable vs value, variable vs variable)

Checklist
- [ ] `EXPRESSION` keyword added and counted
- [ ] `TExpression` and storage added
- [ ] `mathexpr` module integrated
- [ ] `controls_addExpression` implemented
- [ ] Premise parsing recognizes expressions
- [ ] Premise evaluation calls `mathexpr_eval`
- [ ] Tests pass

### Phase C — Simulation Integration & Reporting
- Ensure the rule evaluation cadence matches EPANET’s control step; add or reuse a `RULE_STEP` if applicable
- Apply actions with priority and avoid duplicate actions
- Update reporting/logging to reflect control actions (optional)

Checklist
- [ ] Rule evaluation invoked at correct timestep
- [ ] Actions applied deterministically with priority
- [ ] Optional: report control actions

### Phase D — INP I/O Considerations
- EPANET may not write back `[CONTROLS]` from the engine; confirm requirements
- If writing is needed, add a simple serializer for `VARIABLE` and `EXPRESSION` lines in any EPANET writer component (if present)

### Error Handling & Compatibility
- Mirror SWMM errors: invalid keywords, bad names, math expr compile failure, mixed-attribute warning
- Preserve backward compatibility with legacy `[CONTROLS]` syntax

### Test Matrix (examples)
- VARIABLE definitions including `SIMULATION TIME`, `DATE`, `DAY`, `MONTH`, `DAYOFYEAR`
- Expressions: arithmetic, trig, hyperbolic, `log`, `log10`, `exp`, `abs`, `sgn`, `step`, unary minus, exponentiation `^`
- Rule clauses: expression vs value, expression vs variable, variable vs value, variable vs variable
- Edge cases: undefined variable/expression, illegal function names, mismatched attributes, divide by zero (expect 0 result per SWMM behavior)

### Notes
- Keep names and behavior close to SWMM for maintainability
- Prefer minimal changes to EPANET’s existing rule engine; augment rather than replace