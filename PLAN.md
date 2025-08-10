### Goal
Port SWMM 5.2 "Named Variables" and "Arithmetic Expressions" control-rule features into EPANET, with clear phases, tests, and documentation.

### Deliverables
- Research Notes: detailed inventory of SWMM 5.2 implementation (files, structs, functions, I/O, runtime), with code citations
- EPANET Integration Guide: phased design and implementation steps with checklists and tests
- Worklog: decisions, assumptions, and risks

### High-Level Phases
1) Discovery and Documentation
- Identify exact SWMM files, structs, enums, and functions for variables/expressions and rule evaluation
- Map INP reading and any writing/reporting behaviors
- Trace runtime usage in the simulation loop

2) EPANET Architectural Mapping
- Review EPANET control rules parsing and evaluation
- Identify integration points: parser, rule store, evaluation, reporting

3) Implementation – Part A: Named Variables
- Add data structures, counters, and storage
- Extend INP parsing to read VARIABLE definitions
- Resolve variables during rule parsing and evaluation
- Unit tests and sample INP

4) Implementation – Part B: Arithmetic Expressions
- Integrate math expression parser module (ported or reused)
- Extend INP parsing to read EXPRESSION definitions
- Evaluate expressions in rule premises via named variables
- Unit tests covering supported functions and operators

5) Integration and Regression Tests
- Simulation loop integration points; ensure rule evaluation cadence is preserved
- Backward compatibility tests on existing EPANET models

6) Documentation
- Update user docs with syntax, examples, and limitations
- Developer docs for maintenance

### Milestone Checklist
- [ ] SWMM code inventory completed and reviewed
- [ ] EPANET control-rule architecture documented
- [ ] Named Variables parsed and stored in EPANET
- [ ] Named Variables used in rule evaluation
- [ ] Arithmetic Expressions parsed and stored in EPANET
- [ ] Expression evaluator integrated and tested
- [ ] INP read/write/report behavior updated
- [ ] End-to-end tests passing

### Constraints & Assumptions
- Maintain EPANET INP compatibility; additions should be opt-in
- Error handling consistent with EPANET conventions
- Minimal performance overhead during rule evaluation