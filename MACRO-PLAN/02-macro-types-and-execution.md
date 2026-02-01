# Macro Types + Execution Model

## Macro Types

1. **User-Defined Functions (UDFs)**
- Pure functions used in formulas (e.g., `=DOUBLE(A1)`).
- Excel analogy: UDFs written in VBA or Office Scripts.
- Must run in the server worker to be authoritative.
- Client can register for UX/autocomplete only.

2. **Macros (Action Macros)**
- Scripted actions that manipulate the workbook (set values, formulas, formats, etc.).
- Excel analogy: VBA macros that edit cells and formatting.
- Implemented as SocialCalc commands and run in the server worker.

## Execution Contexts

- **Server-side** (authoritative): `src/sc.ls` worker runs SocialCalc, applies commands, persists snapshots.
- **Client-side** (UX): browser exposes function list, macro editor, and run UI.

## Execution Flow (High-Level)

1. Macro definitions loaded into worker when a room (sheet/session) initializes.
2. UDFs registered into SocialCalc formula function registry.
3. Action macros executed by passing commands into `ScheduleSheetCommands` or `ExecuteCommand`.
4. Resulting snapshot/log updates propagate to clients.
