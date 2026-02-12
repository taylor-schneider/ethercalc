# Debugging Implementation

## Table of Contents
- [Debugging Implementation](#debugging-implementation)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
    - [Key Libraries and Integrations](#key-libraries-and-integrations)
  - [Changes Required To Implement Desired Functionality](#changes-required-to-implement-desired-functionality)
    - [Referenced EtherCalc Files and Functions (What They Do)](#referenced-ethercalc-files-and-functions-what-they-do)
    - [Change Inventory](#change-inventory)
  - [End-to-End Debugging Workflow](#end-to-end-debugging-workflow)
    - [Workflow Steps](#workflow-steps)
    - [ASCII Diagram](#ascii-diagram)
  - [Debug Protocol](#debug-protocol)
    - [DAP Messages](#dap-messages)
    - [Message Format](#message-format)
  - [Security Considerations](#security-considerations)
  - [Implementation Steps](#implementation-steps)
  - [Dependencies](#dependencies)

## Overview

Macro debugging in EtherCalc enables developers to set breakpoints, step through code, inspect variables, and view call stacks during macro execution. This provides an IDE-like experience in Development Mode, integrating Monaco Editor with custom debugging UI components and the Debug Adapter Protocol (DAP).

Support extends to User-Defined Functions (UDFs), allowing debugging of functions called from spreadsheet formulas. UDF debugging triggers when a formula evaluates a UDF with breakpoints set.

The implementation copies VS Code's Node.js debugger approach: instrument code with `debugger;` statements, execute in Node.js worker threads with inspector enabled, and use DAP for client-server communication.

Key components:
- **Client**: Monaco Editor (with breakpoint gutter) integrated with custom debug toolbar and panels (variable, call stack, console, watch).
- **Server**: Instrumented macro/UDF execution in Node.js worker threads with inspector, using DAP for communication.
- **Protocol**: Debug Adapter Protocol (DAP) over WebSocket for stepping, breakpoints, and state inspection.

Debugging is sandboxed and policy-controlled, with timeouts and resource limits to prevent abuse.

### Key Libraries and Integrations
- **Client-Side Libraries**:
  - **Editor**:
    - **Monaco Editor** (@microsoft/monaco-editor):
      - **Out-of-the-Box**: Code editor, breakpoint gutter, DAP integration API.
    - **Custom Code**: 
      - Debug toolbar 
      - variable panel 
      - call stack panel 
      - console panel
      - watch panel
  - **Communication**:
    - **WebSocket API**: Built-in browser API for real-time DAP communication with the server.
- **Server-Side Libraries**:
  - **Debug Protocol**:
    - **vscode-debugadapter**: Implements the DAP server, translating DAP requests/responses to/from Node inspector commands.
  - **Execution Environment**:
    - **Node.js worker_threads**: Provides sandboxed, isolated threads for running instrumented macros/UDFs without affecting the main process.
    - **Node.js inspector**: Built-in Node.js module for debugging; handles pausing/resuming execution and inspecting state via WebSocket.
  - **Code Instrumentation**:
    - **Acorn**: Parses JavaScript source code into an Abstract Syntax Tree (AST) for analysis and modification.
    - **Estraverse**: Traverses the AST to insert `debugger;` statements at breakpoint locations.
    - **Escodegen**: Regenerates executable JavaScript code from the modified AST.
  - **Formula Engine**:
    - **SocialCalc**: Spreadsheet formula evaluator; hooked to detect UDF calls and trigger debug sessions with cell context.

## Changes Required To Implement Desired Functionality

EtherCalc's debugging implementation extends the existing client-server spreadsheet architecture to support interactive macro and UDF debugging. At a high level, this involves integrating Monaco Editor for source-level debugging UI on the client, creating a new Debug Adapter module on the server to handle DAP protocol communication, and instrumenting macro/UDF execution in isolated worker threads with Node.js inspector for runtime control.

The key integration points are:

- **Client-Side (Browser)**: Monaco Editor provides out-of-the-box breakpoint management and decorations. Custom debug toolbar and panels (variables, call stack, console, watch) are added to EtherCalc's UI, connected via WebSocket to the server. Client events are dispatched through src/player.ls's broadcast and socket handling.

- **Server-Side (Node.js)**: A new Debug Adapter module translates DAP requests (launch, step, evaluate) into Node inspector commands. It orchestrates instrumented code execution in worker threads, emitting DAP events back to the client. Server routing in src/main.ls forwards debug commands to the adapter.

- **Execution Environment**: Macros/UDFs are instrumented with `debugger;` statements using Acorn/Estraverse/Escodegen. Execution occurs in worker_threads with inspector enabled, controlled via src/sc.ls's worker message switch.

- **Protocol**: DAP over WebSocket ensures secure, room-scoped communication.

This architecture ensures debugging is sandboxed, policy-controlled, and integrates seamlessly with EtherCalc's collaborative editing model.

### Referenced EtherCalc Files and Functions (What They Do)
- **[src/player.ls](../src/player.ls)**
  - Main browser/client collaboration runtime for a sheet room.
  - Defines `SocialCalc.Callbacks.broadcast` (client outbound event path).
  - Contains socket `@on data` dispatch logic that applies incoming events to UI/editor state.
  - Current integration role: primary client-side insertion point for debug UI event handling and DAP event fan-out.
- **[src/main.ls](../src/main.ls)**
  - HTTP API/router layer for room operations and command ingestion.
  - Contains `@post '/_/:room'` for command submission and `SC[room]?ExecuteCommand cmdstr` dispatch.
  - Current integration role: server entry point for debug-session control requests and room-scoped routing.
- **[src/sc.ls](../src/sc.ls)**
  - Server-side sheet worker orchestration and SocialCalc execution bridge.
  - Defines `SC._init(...)` worker bootstrapping and worker `self.onmessage` command switch (`\cmd`, `\recalc`, etc.).
  - Exposes `w.ExecuteCommand(...)` and snapshot/recalc wiring used by the rest of the system.
  - Current integration role: concrete execution-layer extension point for debugger control messages and worker debug hooks.

### Change Inventory

**Column meaning**
- **EtherCalc change set**: changes in EtherCalc client/server files that connect UI actions and room events to debugging.
- **Monaco change set**: changes in Monaco editor API usage to reflect and navigate debug state.
- **Debug Adapter change set**: planned creation work for a **new adapter module** (request handlers, event handlers, inspector mapping). This is **not** changing the DAP specification.


| Custom Component | EtherCalc change set | Monaco change set | Debug Adapter change set |
|---|---|---|---|
| **Debug Toolbar** | <ul><li><strong>EC-TB-01</strong><ul><li>Target: <code>src/player.ls</code> → <code>SocialCalc.Callbacks.broadcast</code></li><li>Issue: toolbar debug actions are not emitted with an explicit debug action shape, so downstream handling is ambiguous.</li><li>Change: add explicit toolbar debug action dispatch (<code>run/debug/step/continue/stop</code>) in the broadcast path.</li><li>Dependencies: requires <code>DA-TB-01</code> and <code>DA-TB-02</code>.</li></ul></li><li><strong>EC-TB-02</strong><ul><li>Target: <code>src/player.ls</code> → socket <code>@on data</code></li><li>Issue: incoming room events do not drive a dedicated toolbar debug state model.</li><li>Change: add debug-state event handling for toolbar enable/disable and mode updates.</li><li>Dependencies: requires <code>DA-TB-03</code>.</li></ul></li><li><strong>EC-TB-03</strong><ul><li>Target: <code>src/main.ls</code> → <code>@post '/_/:room'</code></li><li>Issue: debug control payloads do not have explicit routing into debug handling paths.</li><li>Change: add room endpoint routing rules that forward debug controls into adapter-facing logic.</li><li>Dependencies: requires <code>DA-TB-01</code> and <code>DA-TB-02</code>.</li></ul></li></ul> | <ul><li><strong>MN-TB-01</strong><ul><li>Target: <code>editor.getModel</code></li><li>Issue: launch flow may not resolve the active model deterministically.</li><li>Change: resolve and validate active model before launch payload construction.</li><li>Dependencies: required by <code>EC-TB-01</code>, paired with <code>MN-TB-02</code>.</li></ul></li><li><strong>MN-TB-02</strong><ul><li>Target: <code>model.getValue</code></li><li>Issue: launch can use stale source snapshots.</li><li>Change: read source directly from active model at launch dispatch.</li><li>Dependencies: required by <code>EC-TB-01</code>, used with <code>DA-TB-01</code>.</li></ul></li><li><strong>MN-TB-03</strong><ul><li>Target: <code>editor.deltaDecorations</code></li><li>Issue: run/pause state is not clearly visible in editor view.</li><li>Change: apply explicit debug-state decorations on lifecycle transitions.</li><li>Dependencies: depends on adapter lifecycle events from <code>DA-TB-03</code>.</li></ul></li><li><strong>MN-TB-04</strong><ul><li>Target: <code>editor.revealLineInCenter</code></li><li>Issue: step/continue can move execution outside viewport.</li><li>Change: center execution line after state transitions.</li><li>Dependencies: depends on execution-position updates from <code>DA-TB-03</code>.</li></ul></li></ul> | <ul><li><strong>DA-TB-01</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>launch</code></li><li>Issue: no adapter module exists yet, so there is no canonical launch handling path from toolbar actions.</li><li>Change: create the adapter module and implement a <code>launch</code> handler that initializes a debug session for macro/UDF execution.</li><li>Dependencies: required by <code>EC-TB-01</code>, <code>EC-TB-03</code>, <code>MN-TB-01</code>, <code>MN-TB-02</code>.</li></ul></li><li><strong>DA-TB-02</strong><ul><li>Target: new Debug Adapter module (planned) → request handlers for <code>continue</code>, <code>next</code>, <code>stepIn</code>, <code>stepOut</code>, <code>terminate</code></li><li>Issue: toolbar controls have no backend adapter handlers, so step/resume/stop actions cannot be translated into runtime operations.</li><li>Change: create request handlers for each control action and map each handler to the corresponding runtime control command.</li><li>Dependencies: required by <code>EC-TB-01</code> and <code>EC-TB-03</code>.</li></ul></li><li><strong>DA-TB-03</strong><ul><li>Target: new Debug Adapter module (planned) → lifecycle event emission for <code>initialized</code>, <code>stopped</code>, <code>continued</code>, <code>terminated</code></li><li>Issue: without adapter-emitted lifecycle events, toolbar state cannot stay synchronized with debugger runtime state.</li><li>Change: emit normalized lifecycle events from the adapter so client toolbar state transitions are deterministic.</li><li>Dependencies: required by <code>EC-TB-02</code>, <code>MN-TB-03</code>, <code>MN-TB-04</code>.</li></ul></li></ul> |
| **Variable Panel** | <ul><li><strong>EC-VAR-01</strong><ul><li>Target: <code>src/player.ls</code> → socket <code>@on data</code></li><li>Issue: variable refresh is not strictly gated by debug pause lifecycle.</li><li>Change: trigger variable panel refresh only from debug stop-related events.</li><li>Dependencies: depends on <code>DA-VAR-01</code>.</li></ul></li><li><strong>EC-VAR-02</strong><ul><li>Target: new variable panel module</li><li>Issue: no dedicated hierarchical variable UI/state exists.</li><li>Change: implement variable tree with expansion-state persistence across steps.</li><li>Dependencies: depends on <code>DA-VAR-02</code> and <code>DA-VAR-03</code>.</li></ul></li></ul> | <ul><li><strong>MN-VAR-01</strong><ul><li>Target: <code>editor.deltaDecorations</code></li><li>Issue: selected variable/scope context is disconnected from source view.</li><li>Change: add optional source decorations tied to selected frame/scope.</li><li>Dependencies: depends on frame context from <code>DA-STK-02</code>.</li></ul></li><li><strong>MN-VAR-02</strong><ul><li>Target: <code>editor.setModelMarkers</code></li><li>Issue: variable resolution failures are not surfaced in editor diagnostics.</li><li>Change: emit markers for variable/evaluation errors when configured.</li><li>Dependencies: depends on error payloads from <code>DA-VAR-03</code> / <code>DA-CON-02</code>.</li></ul></li></ul> | <ul><li><strong>DA-VAR-01</strong><ul><li>Target: new Debug Adapter module (planned) → event handling for <code>stopped</code></li><li>Issue: no adapter module exists yet to gate variable reads on confirmed pause state.</li><li>Change: create stop-event handling in the adapter and trigger variable fetch workflow only after pause is confirmed.</li><li>Dependencies: required by <code>EC-VAR-01</code>.</li></ul></li><li><strong>DA-VAR-02</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>scopes</code></li><li>Issue: scope references cannot be resolved without an adapter-level scopes implementation.</li><li>Change: implement a <code>scopes</code> handler that resolves scope references for the selected frame.</li><li>Dependencies: depends on frame selection from <code>EC-STK-02</code>; required by <code>EC-VAR-02</code>.</li></ul></li><li><strong>DA-VAR-03</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>variables</code></li><li>Issue: variable panel cannot hydrate rows until scope references are translated into variable payloads.</li><li>Change: implement a <code>variables</code> handler that returns structured values for each scope reference.</li><li>Dependencies: depends on <code>DA-VAR-02</code>; required by <code>EC-VAR-02</code>.</li></ul></li></ul> |
| **Call Stack Panel** | <ul><li><strong>EC-STK-01</strong><ul><li>Target: <code>src/player.ls</code> → socket <code>@on data</code></li><li>Issue: stack payload and frame selection events are not first-class in current client dispatch.</li><li>Change: add stack event handling and frame-selection dispatch in socket flow.</li><li>Dependencies: depends on <code>DA-STK-01</code> and <code>DA-STK-02</code>.</li></ul></li><li><strong>EC-STK-02</strong><ul><li>Target: new call-stack module</li><li>Issue: no state container for frame list and active frame identity.</li><li>Change: implement stack module with active <code>frameId</code> persistence and selection API.</li><li>Dependencies: depends on <code>DA-STK-02</code>; required by <code>DA-VAR-02</code>, <code>DA-STK-03</code>, <code>DA-CON-02</code>, <code>DA-WAT-02</code>.</li></ul></li></ul> | <ul><li><strong>MN-STK-01</strong><ul><li>Target: <code>editor.setModel</code></li><li>Issue: selected frame file can differ from current Monaco model.</li><li>Change: switch model when selected frame source changes.</li><li>Dependencies: depends on frame file info from <code>DA-STK-02</code>.</li></ul></li><li><strong>MN-STK-02</strong><ul><li>Target: <code>editor.revealLineInCenter</code></li><li>Issue: selected frame line may remain off-screen.</li><li>Change: reveal and center selected frame line on stack selection.</li><li>Dependencies: depends on frame line info from <code>DA-STK-02</code>.</li></ul></li><li><strong>MN-STK-03</strong><ul><li>Target: <code>editor.deltaDecorations</code></li><li>Issue: active frame line is not visually distinct.</li><li>Change: apply active-frame highlight decoration on selection changes.</li><li>Dependencies: depends on active frame state from <code>EC-STK-02</code>.</li></ul></li></ul> | <ul><li><strong>DA-STK-01</strong><ul><li>Target: new Debug Adapter module (planned) → event handling for <code>stopped</code></li><li>Issue: no adapter implementation exists to orchestrate stack refresh timing from stop lifecycle events.</li><li>Change: create stop-event handling that triggers stack refresh workflow after pause is confirmed.</li><li>Dependencies: required by <code>EC-STK-01</code>.</li></ul></li><li><strong>DA-STK-02</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>stackTrace</code></li><li>Issue: call stack frames cannot be returned without a stackTrace handler in the adapter.</li><li>Change: implement <code>stackTrace</code> handling that converts runtime frames into DAP frame payloads.</li><li>Dependencies: required by <code>EC-STK-01</code>, <code>EC-STK-02</code>, <code>MN-STK-01</code>, <code>MN-STK-02</code>.</li></ul></li><li><strong>DA-STK-03</strong><ul><li>Target: new Debug Adapter module (planned) → request context routing for <code>scopes</code> and <code>evaluate</code></li><li>Issue: selected frame context can be lost across downstream requests without adapter-side routing rules.</li><li>Change: enforce propagation of selected <code>frameId</code> in scope/evaluate request handling paths.</li><li>Dependencies: depends on <code>EC-STK-02</code>; required by <code>DA-VAR-02</code>, <code>DA-CON-02</code>, <code>DA-WAT-02</code>.</li></ul></li></ul> |
| **Console Panel** | <ul><li><strong>EC-CON-01</strong><ul><li>Target: <code>src/player.ls</code> → socket <code>@on data</code></li><li>Issue: debug output events are not normalized into a dedicated console stream in client state.</li><li>Change: add console event normalization and append pipeline in socket processing.</li><li>Dependencies: depends on <code>DA-CON-01</code>.</li></ul></li><li><strong>EC-CON-02</strong><ul><li>Target: new console module</li><li>Issue: no dedicated debug console UI/state exists for stream, input, and history.</li><li>Change: implement console module for output rendering and expression input history.</li><li>Dependencies: depends on <code>DA-CON-01</code> and <code>DA-CON-02</code>.</li></ul></li></ul> | <ul><li><strong>MN-CON-01</strong><ul><li>Target: <code>editor.setModel</code></li><li>Issue: console source links can point to files not currently loaded.</li><li>Change: switch editor model to linked source file before line navigation.</li><li>Dependencies: depends on file/line references from <code>DA-CON-01</code>.</li></ul></li><li><strong>MN-CON-02</strong><ul><li>Target: <code>editor.revealLineInCenter</code></li><li>Issue: linked source line can remain out of view after navigation.</li><li>Change: center linked line when navigating from console output.</li><li>Dependencies: depends on <code>MN-CON-01</code> and source location from <code>DA-CON-01</code>.</li></ul></li></ul> | <ul><li><strong>DA-CON-01</strong><ul><li>Target: new Debug Adapter module (planned) → event emission for <code>output</code></li><li>Issue: no adapter implementation exists to normalize runtime stdout/stderr/debug output into client-consumable events.</li><li>Change: create output-event emission in the adapter with consistent category and payload mapping.</li><li>Dependencies: required by <code>EC-CON-01</code>, <code>EC-CON-02</code>, <code>MN-CON-01</code>, <code>MN-CON-02</code>.</li></ul></li><li><strong>DA-CON-02</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>evaluate</code></li><li>Issue: console expressions cannot be evaluated through a frame-aware adapter path because no evaluate handler exists yet.</li><li>Change: implement an <code>evaluate</code> handler that executes expressions with active frame context and returns normalized success/error responses.</li><li>Dependencies: depends on <code>DA-STK-03</code>; required by <code>EC-CON-02</code>.</li></ul></li></ul> |
| **Watch Panel** | <ul><li><strong>EC-WAT-01</strong><ul><li>Target: <code>src/player.ls</code> pause-event path</li><li>Issue: watch values are not reliably refreshed at every debugger stop.</li><li>Change: trigger watch reevaluation from pause lifecycle events.</li><li>Dependencies: depends on <code>DA-WAT-01</code>.</li></ul></li><li><strong>EC-WAT-02</strong><ul><li>Target: new watch-store module</li><li>Issue: no stable state model exists for watch expression lifecycle and persistence.</li><li>Change: implement watch-store for add/remove/persist behavior per user/session.</li><li>Dependencies: depends on <code>DA-WAT-02</code>.</li></ul></li></ul> | <ul><li><strong>MN-WAT-01</strong><ul><li>Target: Monaco active model/position state</li><li>Issue: watch context can drift from currently viewed source/frame context.</li><li>Change: bind watch evaluation context derivation to active model/position and selected frame.</li><li>Dependencies: depends on <code>EC-STK-02</code> and <code>DA-STK-03</code>.</li></ul></li></ul> | <ul><li><strong>DA-WAT-01</strong><ul><li>Target: new Debug Adapter module (planned) → event handling for <code>stopped</code></li><li>Issue: no adapter exists to synchronize watch reevaluation with confirmed pause lifecycle events.</li><li>Change: create stop-event handling in the adapter that triggers watch reevaluation workflow.</li><li>Dependencies: required by <code>EC-WAT-01</code>.</li></ul></li><li><strong>DA-WAT-02</strong><ul><li>Target: new Debug Adapter module (planned) → request handler for <code>evaluate</code></li><li>Issue: watch expressions cannot be evaluated consistently with frame context until adapter evaluate handling exists.</li><li>Change: implement frame-aware evaluate handling for watch expressions with normalized value/error responses.</li><li>Dependencies: depends on <code>DA-STK-03</code>; required by <code>EC-WAT-02</code>.</li></ul></li></ul> |

**Shared behavior across all custom components**
- **Client session manager (new module)**: owns WebSocket transport, DAP request/response correlation (`seq`/`request_seq`), and fan-out to toolbar/panels.
- **Server adapter (new module)**: receives DAP requests, translates to Node inspector calls, emits DAP events back over room-scoped socket channel.
- **Worker integration (existing extension point)**: `SC._init` message switch in [src/sc.ls](../src/sc.ls) is the concrete place to add debug message types (e.g., attach/step/evaluate control plane).
- **Lifecycle contract**: `launch` initializes UI state, `stopped` refreshes Variables/Stack/Watch, `continued` flips running state, `terminated` clears transient debug state.

**What is OTB vs Custom**
- **Monaco OTB**: editor widget, glyph margin, decorations, markers, model/position navigation APIs.
- **Custom in EtherCalc**: debug session manager, toolbar/panels, DAP transport plumbing, room-auth integration, server adapter wiring, worker debug control messages.

## End-to-End Debugging Workflow

The debugging process involves coordinated actions between the client (browser), server (Node.js), and execution environment (worker thread with inspector). Below is the complete workflow for debugging a macro or UDF.

### Workflow Steps
1. **Breakpoint Setup (Client)**: User clicks in Monaco editor gutter to set breakpoints. Editor stores breakpoints locally and sends them to server via DAP on debug start.
2. **Debug Session Initiation (Client → Server)**: User clicks "Debug" button. Client sends `launch` DAP request with macroId/funcId, breakpoints, and type.
3. **Instrumentation (Server)**: Server uses Acorn/Estraverse to build AST from macro/UDF source. Inserts `debugger;` at breakpoints, regenerates code with Escodegen.
4. **Execution Preparation (Server)**: Launches worker thread with instrumented code and inspector enabled (`--inspect=0`).
5. **Execution Start (Server)**: Worker runs code; DAP adapter attaches to inspector's WebSocket.
6. **Pause Trigger (Server)**: `debugger;` hits, inspector pauses, sends event to DAP adapter.
7. **State Transmission (Server → Client)**: DAP adapter sends `stopped` event with line, variables, stack, context over WebSocket.
8. **UI Update (Client)**: Monaco highlights paused line, updates variable panel, stack panel, console.
9. **User Action (Client)**: User clicks Step/Continue/Stop. Client sends DAP request (e.g., `next`, `continue`).
10. **Resume/Step (Server)**: DAP adapter sends inspector command (e.g., `Debugger.stepOver`), resumes worker.
11. **Loop or Completion**: Repeat steps 6-10 until execution finishes or stopped.
12. **Session End (Server → Client)**: Sends `terminated` DAP event. Client resets UI.

### ASCII Diagram
```
Client (Browser)              Server (Node.js)              Worker Thread
     |                             |                             |
     | 1. Set Breakpoints          |                             |
     |    (Monaco UI)              |                             |
     |---------------------------->|                             |
     |                             | 2. Receive launch           |
     |                             |    Instrument Code          |
     |                             |    Launch Worker            |
     |                             |---------------------------->|
     |                             |                             | 3. Run Instrumented Code
     |                             |                             |    Execute until debugger;
     |                             |<----------------------------|
     |                             | 4. debugger; Triggers Pause |
     |                             |    Inspector Sends Event    |
     |<----------------------------| 5. Send 'stopped'           |
     | 6. Update UI (Highlight,    |                             |
     |     Variables, Stack)       |                             |
     |                             |                             |
     | 7. User Action (Step/Cont)  |                             |
     |---------------------------->| 8. Receive next/continue    |
     |                             |    Send Inspector Command   |
     |                             |---------------------------->|
     |                             |                             | 9. Execute Next
     |                             |                             |    (Loop to 4 on debugger;)
     |                             |                             |
     |                             | 10. On Completion/Error     |
     |<----------------------------| 11. Send 'terminated'       |
     | 12. Reset UI                |                             |
```

For UDFs, steps 1-3 occur on UDF load if breakpoints set; execution triggers during formula eval in SocialCalc, with additional cell context in state.

## Debug Protocol

### DAP Messages
For the full Debug Adapter Protocol specification, refer to [https://microsoft.github.io/debug-adapter-protocol/specification](https://microsoft.github.io/debug-adapter-protocol/specification).

- **Client → Server**:
  - `launch`: { program: macroId|funcId, breakpoints[], type: 'macro'|'udf' }
  - `setBreakpoints`: { source, breakpoints[] }
  - `next`: { threadId }
  - `continue`: { threadId }
  - `stepIn`: { threadId }
  - `stepOut`: { threadId }
  - `evaluate`: { expression, frameId }

- **Server → Client**:
  - `stopped`: { reason: 'breakpoint'|'step', threadId, line, variables?, stack? }
  - `continued`: { threadId }
  - `terminated`: {}
  - `output`: { category: 'stdout'|'stderr', output }

### Message Format
JSON-RPC 2.0 over WebSocket. Secure via room auth and macro policies.

## Security Considerations
- **Sandbox Limits**: Access to host filesystem/network during debug is controlled by server/user/macro security settings, not hardcoded; debugging operates within the same policy constraints as normal execution.
- **Policy Enforcement**: Debug sessions respect macro permissions.
- **Resource Caps**: Debug time, memory, and message rate limits are derived from server/user/macro level settings, not hardcoded.
- **Audit Logging**: Log debug actions for compliance.

## Implementation Steps
1. Integrate Monaco debug API in client bundle; register DAP descriptor factory pointing to server endpoint.
2. Build custom debugging UI components (debug toolbar, variable panel, call stack panel, console panel, watch panel) using HTML/CSS/JS, connected to DAP events.
3. Add DAP server using vscode-debugadapter in backend.
4. Implement instrumentation in macro runner and SocialCalc UDF evaluator using Acorn/Estraverse/Escodegen.
5. Build worker thread launcher with inspector support.
6. Attach DAP adapter to worker inspector WebSocket.
7. Test with sample macros and UDFs; validate security.

## Dependencies
- Monaco Editor integration depends on client bundle updates and Webpack config; add @microsoft/monaco-editor to package.json (provides editor and breakpoint gutter OTB; custom UI components required for full debugging).
- Custom debugging UI depends on HTML/CSS/JS for toolbar and panels; integrate with Monaco's debug API.
- DAP server depends on vscode-debugadapter for protocol handling.
- Instrumentation depends on acorn, estraverse, escodegen for AST manipulation.
- Worker execution depends on Node.js worker_threads and inspector modules (built-in).
- UDF debugging depends on SocialCalc formula engine hooks for breakpoint detection (modify sc.js or related files).
- Debug protocol depends on backend API extensions in [05-backend-api.md](05-backend-api.md).
- Security policies depend on [04-security-permissions.md](04-security-permissions.md) for debug limits.