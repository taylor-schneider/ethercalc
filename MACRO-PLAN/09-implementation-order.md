# Implementation Order

## Table of Contents
- [Remaining todos for planning](#remaining-todos-for-planning)

1. **Workbook-embedded macro storage** + load-on-init in `src/sc.ls` (room = sheet/session).
2. **UDF registration** in server worker + client registry exposure (Excel analogy: custom functions available in formulas).
3. **Macro runner** with command API and limits (Excel analogy: macro execution).
4. **Backend API** endpoints for CRUD/run, repo sync, and locks.
5. **UI** editor/runner + repo link panels.
6. **Optional external repo flow** (guided sync UI + `.macro.js` scanning).

## Remaining todos for planning
- No test plan for macro execution, security limits, and sync in MACRO-PLAN/09-implementation-order.md. **DONE**: Added test checklist below.
- Add a test checklist covering UDFs, macros, locking, repo sync, and security limits. **DONE**.

## Test Checklist
- **UDF Tests**: Register UDF, call in formula, verify output; test arg validation; test policy blocks.
- **Macro Tests**: Execute macro via API/UI, check results; test async execution; verify sandbox limits (runtime, steps).
- **Locking Tests**: Edit lock prevents concurrent edits; unlock releases; timeout expires lock.
- **Repo Sync Tests**: Link repo, sync macros/UDFs; verify commit pinning; test file scanning (.macro.js/.func.js).
- **Security Limits Tests**: Network access blocked; runtime exceeded throws error; step limit enforced.
- **Policy Tests**: Global/room/macro/user hierarchy applied; overrides work; audit logs record changes.
- **Storage Tests**: Macros persist in snapshots; load on init; backward compat with old snapshots.
