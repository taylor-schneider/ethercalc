# Execution Model For Macros and UDFs

## Table of Contents
- [Concepts](#concepts)
- [Execution Contexts](#execution-contexts)
- [Execution Flow (High-Level)](#execution-flow-high-level)
- [Execution Flow Diagram (Boxes + Arrows)](#execution-flow-diagram-boxes--arrows)

## Concepts

**User‑Defined Functions (UDFs)**
- Pure JS functions defined by the user
- Able to be referenced / used in worksheet formulas (e.g., `=DOUBLE(A1)`).
- Excel analogy: UDFs written in VBA or Office Scripts.
- Must run in the server worker to be authoritative; client registration is only for UX/autocomplete.

**Macros**
- Pure JS implementations
- Scripted actions that manipulate the workbook (set values, formulas, formats, etc.).
- Excel analogy: VBA macros that edit cells and formatting.
- Implemented as SocialCalc command strings and executed in the server worker.
- There is no verified built‑in macro registry in the current codebase; we will add a macro API in the worker context.

## Execution Contexts

- **Server‑side (authoritative)**: the `src/sc.ls` worker runs SocialCalc, applies commands, and persists snapshots.
- **Client‑side (UX)**: the browser exposes the function list, macro editor, and run UI.

## Execution Flow (High‑Level)

1. Macro definitions loaded into the worker from storage (embedded snapshot; repo sync updates the embedded macros) when a room (sheet/session) initializes.
2. UDFs registered into the SocialCalc formula registry in the worker during `SC._init` (worker `init` phase) in [src/sc.ls](src/sc.ls#L155-L190), immediately after the snapshot is decoded and before commands are scheduled.
3. Macros executed by passing SocialCalc commands into `ScheduleSheetCommands` or `ExecuteCommand`.
	- **Command execution hooks**: `window.ss.ExecuteCommand` for single commands and `ss.context.sheetobj.ScheduleSheetCommands` for batched commands in [src/sc.ls](src/sc.ls#L130-L190).
	- **Formula registry**: SocialCalc uses `SocialCalc.Formula.FunctionList` during calculation (visible in the bundled runtime) in [static/ethercalc.js](static/ethercalc.js#L9).
4. Resulting snapshot/log updates propagate to clients.

## Execution Flow Diagram (Boxes + Arrows)

```
[Client UI]
	|
	v
[Request: load room]
	|
	v
[Server worker init]
	|
	+--> [Load macros from embedded snapshot]
	|
	+--> [Repo sync (if linked) updates embedded macros]
	|
	+--> [Register UDFs in Formula.FunctionList (registry)]
	|
	v
[Sheet ready]
	|
	+--> [User edits cells] -> [Recalc uses UDFs]
	|
	+--> [Run macro] -> [ExecuteCommand / ScheduleSheetCommands]
	|
	+--> [Macro/UDF updated] -> [Lock check] -> [Reload registry (UDFs)] -> [Recalc]
	|
	v
[Snapshot/log updated] -> [Clients notified]
```

Concurrency note:
- Macro/UDF edits must acquire the edit lock (see MACRO-PLAN/05-backend-api.md). If locked, the editor panel does not open and the client shows the lock owner.
- Successful updates trigger registry reload and a recalculation so formulas using UDFs refresh.
- If updates come from an external repo sync, the UI should show a non‑blocking “Macros updated” banner with a reload/refresh action (see MACRO-PLAN/06-ui-integration.md).
