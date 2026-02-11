# UI Integration

## Table of Contents
- [UI Integration](#ui-integration)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [User Journeys](#user-journeys)
    - [End-User Journey (Macro Execution)](#end-user-journey-macro-execution)
    - [Developer Journey (Macro Creation)](#developer-journey-macro-creation)
  - [UI Architecture](#ui-architecture)
    - [Workspace Modes](#workspace-modes)
    - [Layout Diagrams](#layout-diagrams)
      - [Default Mode Layout](#default-mode-layout)
      - [Development Mode Layout](#development-mode-layout)
  - [Component Details](#component-details)
    - [Menu Items / Entry Points](#menu-items--entry-points)
    - [Macro Workspace Panels](#macro-workspace-panels)
    - [Admin Portal Panels](#admin-portal-panels)
    - [Forms / Modals](#forms--modals)
  - [Implementation Considerations](#implementation-considerations)
    - [Error Handling and Lock Waiting UI](#error-handling-and-lock-waiting-ui)
      - [Error Modals](#error-modals)
      - [Lock Waiting Modals](#lock-waiting-modals)
    - [Macro Settings (Details)](#macro-settings-details)
    - [Formula UX (Details)](#formula-ux-details)
    - [Editing Concurrency (Details)](#editing-concurrency-details)
    - [Guided Git UI (Planned)](#guided-git-ui-planned)
    - [Files Likely Touched](#files-likely-touched)
  - [Remaining todos for planning](#remaining-todos-for-planning)
  - [Remaining todos for planning](#remaining-todos-for-planning-1)

## Overview

The EtherCalc macro system provides two distinct user experiences:

1. **End-User Experience**: Simple macro discovery and execution for spreadsheet users
2. **Developer Experience**: Full IDE-like environment for creating, editing, and debugging macros

The UI is designed to serve both audiences through a single Macro Workspace that adapts based on user needs and toggles between modes.

Note: Function list UI changes for hierarchical UDF support (autocomplete, function dialog) are detailed in MACRO-PLAN/12-populating-function-list.md.

## User Journeys

### End-User Journey (Macro Execution)
1. **Access Macros**: Main Menu → Select "Macros" → Macro Workspace opens in default mode, showing the workbook/worksheet in the main view
2. **Browse & Select**: Use collapsible left sidebar (click/hover to expand) to browse available macros by name, type, or author; resize panels as needed
3. **Execute**: Select macro and run with parameters; view results in bottom panel
4. **Manage**: Adjust settings, view audit logs, or link repos for additional macros

*Target User*: Spreadsheet users who want to leverage existing macros without development

### Developer Journey (Macro Creation)
1. **Enter Development**: Main Menu → Select "Macros" → Toggle "Dev Mode" in workspace
2. **Setup Workspace**: Choose view layout (split code/sheet, code-only, or sheet-only)
3. **Develop Iteratively**: Edit code (left) while seeing live worksheet changes (right)
4. **Test & Debug**: Run macros, view console output, inspect worksheet effects
5. **Commit Changes**: For repo-linked macros, commit with diff preview

*Target User*: Developers creating and maintaining macros

## UI Architecture

### Workspace Modes

**Default Mode** (End-User):
- Compact interface focused on macro discovery and execution
- Simple list view with collapsible sidebars
- No code editing capabilities
- Suitable for users who just want to run macros

**Development Mode** (Developer):
- IDE-like environment with persistent code editor
- Split view between code and live worksheet
- Full debugging and development tools
- Auto-save, syntax checking, and iterative workflow

### Layout Diagrams

#### Default Mode Layout
```
┌─────────────────────────────────────────────────┐
│ Top Toolbar: [Dev Mode Toggle] [Repo Link]      │
├─────────────────┬───────────────────────────────┤
│ Left Sidebar    │ Main Panel                    │
│ (Macro List)    │ (Compact View)                │
│ • Macro A       │                               │
│ • Macro B       │  [Run Macro] [View Results]   │
│ • ...           │                               │
│                 │  Results/Errors Panel         │
│                 │  ──────────────────────────   │
├─────────────────┤                               │
│ Right Sidebar   │                               │
│ (Settings)      │                               │
│ • Permissions   │                               │
│ • Audit Logs    │                               │
└─────────────────┴───────────────────────────────┘
```

#### Development Mode Layout
```
┌─────────────────────────────────────────────────┐
│ Top Toolbar: [Dev Mode ON] [View Controls]      │
├─────────────┬─────────────────┬─────────────────┤
│ Left        │ Code Editor     │ Worksheet       │
│ Sidebar     │ (Persistent)    │ (Live View)     │
│ • Macro     ├─────────────────┤                 │
│   List      │ Run/Debug       │ • Cell changes │
│ • File      │ Console         │ • Formula      │
│   Browser   │ ─────────────   │   updates      │
│             │ [Run] [Debug]   │                 │
├─────────────┴─────────────────┴─────────────────┤
│ Bottom Panel: Expanded Results/Errors Console   │
└─────────────────────────────────────────────────┘
```

*View Controls*: Split View | Code Only | Sheet Only (with resizable divider)

## Component Details

### Menu Items / Entry Points
- **Macros** (main menu): Opens the Macro Workspace (full-page UI) for macro discovery, execution, and management. Toggle Dev Mode for editing/debugging. Excel analogy: Macros dialog.
- **Admin Policies** (admin menu): Opens the Admin Portal (full-page UI) for global/room policies and user overrides. Requires admin auth.

### Macro Workspace Panels
- **Macro List Panel** (collapsible left sidebar in Macro Workspace):
  - **Features**: List/search macros by name, type (UDF/macro), source (embedded/repo), status (locked/running). Filter by author or room. In Dev Mode, acts as file explorer for quick switching.
  - **Actions**: Select for running (default mode) or editing/running (Dev Mode); delete (with confirmation); view details. Double-click opens editor (only in Dev Mode).
  - **Links**: Integrates with GET /room/:id/macros from [05-backend-api.md](05-backend-api.md).

- **Repo File Browser Panel** (integrated into left sidebar when repo linked; collapsible):
  - **Features**: Tree view of repo structure; highlights `.macro.js` and `.func.js` files. Shows sync status per file (synced/modified/local-only).
  - **Actions**: Select file to load/edit; sync individual files; view diff.
  - **Links**: Fetches repo tree via GET /room/:id/macros/repos/:repoId/tree; sync via POST /room/:id/macros/repos/:repoId/sync from [05-backend-api.md](05-backend-api.md).

- **Macro Settings Panel** (collapsible right sidebar in Macro Workspace; tabbed with Audit):
  - **Features**: Display macro permissions, network access, policy summary (macro-level), run limits. Edit macro policies via Policy Editor Modal.
  - **Actions**: Open Policy Editor; view effective policies.
  - **Links**: Uses GET /policies/macro/* from [05-backend-api.md](05-backend-api.md); ties to [04-security-permissions.md](04-security-permissions.md).

- **Repo Link Panel** (top toolbar in Macro Workspace; collapsible settings section):
  - **Features**: Input repo URL, select branch/commit, view sync status. Triggers file browser on link.
  - **Actions**: Link repo, sync all (calls POST /room/:id/macros/repos/:repoId/sync).
  - **Links**: [05-backend-api.md](05-backend-api.md) for repo endpoints.

- **Audit Panel** (right sidebar tab in Macro Workspace; collapsible):
  - **Features**: List macro run history, errors, and timestamps.
  - **Actions**: Filter by date; export logs.
  - **Links**: Reads from DB audit logs (e.g., "audit-" + room).

- **Results/Errors Panel** (collapsible bottom panel in Macro Workspace; expands in Dev Mode for debug console):
  - **Features**: Display last run output/errors with collapsible stack trace. In Dev Mode, live console for stdout/stderr.
  - **Actions**: Retry run; clear history; filter logs.
  - **Links**: Updates via websocket or API responses.

### Admin Portal Panels
- **Global Policy Panel** (corresponds to "Global/Room Policies"):
  - **Features**: Edit server-wide defaults (network, timeouts); preview affected rooms/macros.
  - **Actions**: Save (PUT /policies/global); reset to defaults.
  - **Links**: [11-sandboxing.md](11-sandboxing.md) for config; [05-backend-api.md](05-backend-api.md).

- **Room Policy Panel** (corresponds to "Global/Room Policies"):
  - **Features**: List rooms; edit per-room overrides.
  - **Actions**: Apply to selected rooms; audit changes.
  - **Links**: PUT /policies/room/*; requires admin auth per [07-backend-api-security.md](07-backend-api-security.md).

- **User Overrides Dashboard** (corresponds to "User Overrides Dashboard"):
  - **Features**: List pending/approved overrides; show scope (room/macro/global), expiry.
  - **Actions**: Approve/deny requests; revoke active overrides.
  - **Links**: GET/PUT /policies/user/*; audits to DB.

- **Audit Logs Panel** (corresponds to "Audit Logs"):
  - **Features**: Search/filter logs by user, action, date; export.
  - **Actions**: View details; flag suspicious activity.
  - **Links**: Reads from DB audit tables.

### Forms / Modals
- **Macro Editor Modal** (modal overlay; corresponds to "UDF/Macro Editor"):
  - **Features**: Code editor with syntax highlighting, save/discard buttons. Preview UDF signature for functions.
  - **Validation**: AST check; name uniqueness.
  - **Actions**: Save (POST/PUT /room/:id/macros), test run (POST /room/:id/macros/:macroId/run).
  - **Links**: [05-backend-api.md](05-backend-api.md) for CRUD; integrates with SocialCalc editor.

- **Run Confirmation Modal** (modal overlay; corresponds to "Run Dialog"):
  - **Features**: Display macro details, required permissions, potential risks. Checkbox for "I understand the risks".
  - **Validation**: Arg types; policy checks.
  - **Actions**: Confirm run (POST /room/:id/macros/:macroId/run); cancel.
  - **Links**: Ties to [04-security-permissions.md](04-security-permissions.md) for risk assessment.

- **Repo Sync Modal** (modal overlay):
  - **Features**: Confirm sync; show diff preview.
  - **Actions**: Proceed (POST /room/:id/macros/repos/:repoId/sync); cancel.
  - **Links**: [05-backend-api.md](05-backend-api.md) for repo endpoints.

- **Policy Editor Modal** (modal overlay):
  - **Features**: Form for editing macro/room/user policies (permissions, network, limits). Hierarchical display (global > room > macro > user).
  - **Validation**: Hierarchy checks (can't loosen); real-time feedback.
  - **Actions**: Save policies (PUT /policies/*); requires lock.
  - **Links**: [04-security-permissions.md](04-security-permissions.md) for policy structure; [05-backend-api.md](05-backend-api.md) for endpoints.

- **Override Request Modal** (modal overlay):
  - **Features**: Reason field; select scope.
  - **Actions**: Submit request.
  - **Links**: Triggers POST to override endpoint (if implemented).

- **Import/Export Modal** (modal overlay):
  - **Features**: Upload/download macro files (.macro.js/.func.js). Preview before import.
  - **Validation**: File type check; preview content.
  - **Actions**: Import (POST /room/:id/macros/import); export (GET /room/:id/macros/:macroId/export).
  - **Links**: [05-backend-api.md](05-backend-api.md) for import/export endpoints.

- **Error Details Modal** (modal overlay):
  - **Features**: Full error stack trace, line highlighting in code editor.
  - **Actions**: Jump to line in editor; report issue.
  - **Links**: Displays data from Results/Errors Panel.

- **Lock Waiting Modal** (modal overlay):
  - **Features**: Lock holder, time remaining.
  - **Actions**: Wait, request unlock, cancel.
  - **Links**: Lock endpoints in [05-backend-api.md](05-backend-api.md).

## Implementation Considerations

### Error Handling and Lock Waiting UI

#### Error Modals
- **Triggered By**: API failures (e.g., 403 Permission denied, 409 Conflict, 423 Locked), validation errors, or policy conflicts.
- **Display**: Modal with error message (e.g., "Macro execution blocked: network access not allowed."), details, and stack trace if available.
- **Actions**: Retry (if applicable), Request Override (links to override modal), Cancel, or View Logs.
- **Integration**: Tied to API responses from [05-backend-api.md](05-backend-api.md); shows user-friendly messages from [04-security-permissions.md](04-security-permissions.md) examples.

#### Lock Waiting Modals
- **Triggered By**: Attempting to edit a locked macro/policy (423 Locked response).
- **Display**: Modal showing "Resource locked by [user] since [time]. Expires in [X] minutes."
- **Actions**: Wait (auto-refresh), Request Unlock (sends notification to holder), Cancel.
- **Resolution**: Auto-dismiss when lock expires or is released; updates via websocket polling.
- **Links**: Uses lock endpoints from [05-backend-api.md](05-backend-api.md); concurrency details in editing section above.

### Macro Settings (Details)
- **Scope**: global / room / macro overrides (read-only where not editable).
- **Capabilities**: network access toggle (if allowed), execution time limit, step limit.
- **Permissions**: who can edit/run (admin-only, owners, collaborators).
- **Repo linkage**: show associated `repoId` and pinned commit if sourced from repo.
- **Audit**: last run status, last editor, last sync.

### Formula UX (Details)
- Show UDF names in the formula input area (cell edit mode and the formula bar).
- Include short descriptions and argument hints as the user types.

### Editing Concurrency (Details)
- Lock a macro during edit; other users see who is editing and can request unlock if authorized.
- Requires backend lock endpoints and storage (MACRO-PLAN/05-backend-api.md).
- If a macro/UDF changes while a user is viewing the editor, show a non‑blocking banner with the new version info and a "Reload" action.

### Guided Git UI (Planned)
- Simple flow with buttons: Connect Repo, Select Branch, Pull, Commit, Push, Cherry-pick.
- Branch selection required; diffing and advanced workflows handled by external VCS hosts (e.g., GitHub).
- Scans repo for `.macro.js` (macros) and `.func.js` (UDFs) files on sync.
- No embedded terminal in the initial version.

### Files Likely Touched
- Client bundle (e.g., `static/ethercalc.js` or build pipeline).
- SocialCalc tab definitions.

## Remaining todos for planning
- UI editor technology choice (Monaco vs CodeMirror) is not locked in MACRO-PLAN/06-ui-integration.md. **NOTE**: Implementation decision; planning complete.
- UI elements (macro panels, formula autocomplete) may not align with current client architecture. **NOTE**: Align during implementation; planning specs are ready.
- Choose editor technology and document integration steps. **NOTE**: Post-planning task.
- Macro development workflow (edit-run-debug-commit) components integration. **DONE**: Added detailed workflow with Dev Mode, persistent editor, run/debug panel, worksheet integration, and commit/sync.
- Separate list view vs. IDE integration. **DONE**: Integrated list as collapsible sidebar in Dev Mode; added repo file browser.
- Define Development Mode and toggle mechanism. **DONE**: Added "What is Development Mode?" section with clear definition, toggle location, and how-to-switch instructions.
- Worksheet integration in Dev Mode. **DONE**: Changed from "preview" to full worksheet view with split controls (side-by-side, code-only, sheet-only).
- Update panel descriptions for collapsibility and Dev Mode behavior. **DONE**: Marked sidebars and panels as collapsible; updated locations and behaviors for Dev Mode vs default mode.


## Remaining todos for planning
- UI editor technology choice (Monaco vs CodeMirror) is not locked in MACRO-PLAN/06-ui-integration.md. **NOTE**: Implementation decision; planning complete.
- UI elements (macro panels, formula autocomplete) may not align with current client architecture. **NOTE**: Align during implementation; planning specs are ready.
- Choose editor technology and document integration steps. **NOTE**: Post-planning task.
- Macro development workflow (edit-run-debug-commit) components integration. **DONE**: Added detailed workflow with Dev Mode, persistent editor, run/debug panel, worksheet integration, and commit/sync.
- Separate list view vs. IDE integration. **DONE**: Integrated list as collapsible sidebar in Dev Mode; added repo file browser.
- Define Development Mode and toggle mechanism. **DONE**: Added "What is Development Mode?" section with clear definition, toggle location, and how-to-switch instructions.
- Worksheet integration in Dev Mode. **DONE**: Changed from "preview" to full worksheet view with split controls (side-by-side, code-only, sheet-only).
- Update panel descriptions for collapsibility and Dev Mode behavior. **DONE**: Marked sidebars and panels as collapsible; updated locations and behaviors for Dev Mode vs default mode.
