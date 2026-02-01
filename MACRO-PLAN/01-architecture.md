# Architecture Overview

EtherCalc is built on **SocialCalc**, a JavaScript spreadsheet engine. Key components:

1. **Formula Engine** - Located in `formula1.js`, handles formula parsing and execution (Excel analogy: built-in formula evaluator).
2. **Sheet Commands** - Commands are executed via `ScheduleSheetCommands` to manipulate cells (Excel analogy: actions a macro would perform).
3. **Function System** - Functions are defined and registered in the formula engine (Excel analogy: built-in functions + UDFs).
4. **Cell Model** - Cells contain formulas, values, formatting stored in `sheet.cells`.
5. **Server Recalc Worker** - EtherCalc runs SocialCalc inside a server-side worker (see `src/sc.ls`), which executes commands and recalculates snapshots.
6. **Snapshot + Log Persistence** - Sheets are persisted as snapshots and logs in Redis (see `src/sc.ls` / `db.js`).

Terminology:
- **Room** = a sheet/session ID (the URL path). In Excel terms, this is the workbook/session you open.
- **Snapshot** = current saved state (Excel analogy: the file contents).
- **Log** = command history (Excel analogy: a macro-like action log).

Notes:
- **Where calculations happen**: the server recalculates formulas and macros; the browser mainly displays results and sends edits.
- **What this implies**: if the browser and server disagree, the server’s result wins and is broadcast to all clients.
- **Why it matters for macros/UDFs**: macros and UDFs must be loaded in the server worker so recalculation is consistent and persisted.
- **EtherCalc architecture**: this server‑authoritative model is how EtherCalc works today; the client does not perform the final recalculation. This is to gaurd against intances where multiple clients reference the same sheet; it ensures consistency because it does not trust the client runtime.

