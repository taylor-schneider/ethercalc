# User Repo Concept

## Table of Contents
- [User Repo Concept](#user-repo-concept)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
    - [Repo Sharing](#repo-sharing)
  - [Repo Types](#repo-types)
    - [1. Snapshot-Embedded Repos](#1-snapshot-embedded-repos)
    - [2. External VCS Repos](#2-external-vcs-repos)
      - [VCS Platform Support](#vcs-platform-support)
      - [User-Provided Repos](#user-provided-repos)
      - [Server-Provided Repos](#server-provided-repos)
  - [Multi-Repo Support](#multi-repo-support)
  - [Credential Management](#credential-management)
  - [Repos Are Lazy Loaded](#repos-are-lazy-loaded)
    - [What Loading Means in EtherCalc](#what-loading-means-in-ethercalc)
    - [Step 1: JavaScript Retrieval](#step-1-javascript-retrieval)
    - [Step 2: JavaScript Loading \& Registration](#step-2-javascript-loading--registration)
    - [Step 3: Execution](#step-3-execution)
      - [Function Execution (UDFs)](#function-execution-udfs)
      - [Macro Execution](#macro-execution)
    - [Validation and Security](#validation-and-security)
    - [UX Triggers for Loading](#ux-triggers-for-loading)
  - [Stateful Executions](#stateful-executions)
  - [Additional Security Considerations](#additional-security-considerations)
  - [Handling Asynchronous Repo Updates](#handling-asynchronous-repo-updates)
  - [Todos](#todos)
  - [References](#references)
  - [UI Components](#ui-components)
    - [Component 1: Add User-Provided External VCS Repo](#component-1-add-user-provided-external-vcs-repo)
    - [Component 2: Create Server-Provided External VCS Repo](#component-2-create-server-provided-external-vcs-repo)
    - [Component 3: Configure Multiple Repos in Worksheet](#component-3-configure-multiple-repos-in-worksheet)
  - [UI Modifications](#ui-modifications)
    - [Modification 1: Fucntion resolution in cell reference](#modification-1-fucntion-resolution-in-cell-reference)
  - [Handling Asynchronous Repo Updates](#handling-asynchronous-repo-updates)

## Overview
A "user repo" is a configurable source of macros and user-defined functions (UDFs) that worksheets can reference. Repos allow macros to be stored, versioned, and shared across worksheets while maintaining security through credential management and policy controls. Worksheets can reference multiple repos, enabling functions from different sources to be combined.

### Repo Sharing
Repos (external not default) can be shared across worksheets by manually configuring the same repo (name/URL) in multiple worksheets. No automated sharing mechanism exists; users must coordinate repo details externally.

## Repo Types

### 1. Snapshot-Embedded Repos
- **Description**: Macros/UDFs stored directly within the worksheet's snapshot.
- **Storage**: Embedded in the `--EtherCalc-Macros--` section of the snapshot.
- **Use Case**: Simple, self-contained macros that travel with the worksheet.
- **Credentials**: None required.
- **Sharing**: Macros are part of the snapshot and shared when the worksheet is copied.

### 2. External VCS Repos
External repos connect to remote version control systems (GitHub, GitLab, etc.) with two flavors:

#### VCS Platform Support
- **Modularity**: VCS integration is designed to be extensible via plugins/modules.
- **Supported Operations**: Clone, fetch, checkout commits/tags, push (for server-provided repos).
- **Platforms**: Initially support GitHub/GitLab, with extension points for others.
- **Implementation**: Deferred to later phases; current design assumes git-based VCS.

#### User-Provided Repos
- **Description**: Users connect their own existing VCS repos by providing credentials.
- **Credential Handling**: Users input credentials (tokens, username/password) when configuring the repo. Credentials are encrypted and stored in the snapshot.
- **Repo Management**: Users manage their own repos externally.
- **Use Case**: Power users with existing macro libraries.
- **Security**: User-managed credentials; server proxies git operations without storing secrets.

#### Server-Provided Repos
- **Description**: Server creates and manages repos on the VCS using its own account/credentials.
- **Credential Handling**: Server admin configures VCS credentials server-side. Users don't provide credentials.
- **Repo Creation**: Server creates repos on-demand (e.g., `ethercalc-macros-{roomId}`) when requested via UI or API.
- **Repo Management**: Server owns and controls the repos.
- **Use Case**: Casual users or enterprise deployments needing controlled macro storage.
- **Sharing**: Repos can be shared across worksheets if the repo name/URL is known and configured.
- **Security**: Server controls all access; no user credential exposure.
External repos connect to remote version control systems (GitHub, GitLab, etc.) with two flavors:



## Multi-Repo Support
- **Repo References**: Worksheets can reference multiple repos of any type.
- **Function Resolution**: UDFs are called with optional repo prefixes in formulas (e.g., `myrepo.MYUDF(A1)`, `default.MYFUNC()` for default repo). Macros are executed separately via API endpoints and do not appear in formulas.
- **Conflict Resolution**: If multiple repos define the same UDF name, explicit prefixes are required. Macros are identified by unique IDs and don't conflict in formulas.
- **Loading**: 


## Credential Management

Each external repo configuration will include an optional credential setting. Some repos may be public or r/o and we may not need or want to perform writes.

This modification weill augment ethercalc to support two types of credentials:

  - **User-Provided Credentials**: Configured per repo by the user when adding a user-provided external VCS repo. These are specific to individual repos and allow users to connect their own personal or team-managed VCS accounts. 
    - Intent: Enable flexible, user-controlled access to external repos without server admin involvement. 
    - Usage pattern: Power users or teams with existing VCS infrastructure who want to integrate their macro libraries directly.
    - Storage: User-provided credentials are encrypted in snapshots; server credentials are environment variables.
    - Encryption: User credentials use snapshot-level encryption. The server secret key will
     encrype them and they will be stored in the snapshot.Credentials are decrypted in memory only during git operations.


  - **Server-Level Credentials**: Configured globally by the server admin for server-provided repos. These are shared across all server-provided repos and managed centrally. Intent: Provide controlled, enterprise-grade repo creation and management without exposing credentials to users. Usage pattern: Organizations needing standardized macro storage or casual users who don't manage their own VCS accounts.


## Repos Are Lazy Loaded

Repos are loaded on-demand to optimize performance and resource usage. Loading occurs only when necessary, avoiding upfront costs for unused repos. Once loaded, repos are cached in memory for the session to prevent redundant operations.

### What Loading Means in EtherCalc

EtherCalc is a Node.js-based web application that executes macros and UDFs in a server-side JavaScript environment. Loading a macro or UDF means preparing it for execution without actually running the code, ensuring UDFs are available when called from formulas and macros are available for API-triggered execution. The "thing" executing functions is the Node.js VM (via V8 engine). The functions themselves are stored in memory in the Node.js process via an object called the execution context. Functions can be dynamically registered or removed from this execution context. Registering with the execution context ensures functions and macros are immediately available in memory for the Node.js process serving that room.

### Step 1: JavaScript Retrieval

We add an internal method to ethercalc that retrieves the raw javascript code from the repo so that it can be eventually loaded and executed.

- **Snapshot-Embedded Repos**: Macros are stored as JSON in the snapshot. Loading parses the JSON, extracts JavaScript code strings.
- **External VCS Repos**: On repo creation, the git repo is cloned to a temporary server directory. The repo, like any git repository, needs to be manually updated via a pull/fetch to update the local source code. Loading scans the repo directory for macro files (files with the file extension specified in this documentation) and extracts the JavaScript code into strings.

### Step 2: JavaScript Loading & Registration

The retrieved raw js code is linted to validate syntax, and then registered in the rooms execution context.

When parsing and registering the functions in the room's execution context (a shared JavaScript object/Map), we can add a non-enumerable property to each function object for differentiation:

```javascript
// When loading a .macro.js file
const macroFunc = eval(codeFromFile);  // or similar safe eval
macroFunc.__ethercalc_type = 'macro';
macroFunc.__ethercalc_id = 'myMacro';  // from filename
// Register in SocialCalc.FunctionList or room context

// Similarly for .func.js
const udfFunc = eval(codeFromFile);
udfFunc.__ethercalc_type = 'udf';
udfFunc.__ethercalc_args = ['arg1', 'arg2'];  // if defined in file
```

This allows runtime inspection in the VM (e.g., `if (func.__ethercalc_type === 'macro') { ... }`), while keeping the functions as plain JavaScript.

The loaded javascript fucntions can then be registered

**Registration Separation:**

UDFs and macros are registered in different locations due to their fundamentally different execution models:

- **UDFs → `SocialCalc.Formula.FunctionList`**: 
  - UDFs MUST be registered here because SocialCalc's formula evaluator looks for functions in this registry during formula evaluation
  - When a cell contains `=MYFUNC(A1)`, SocialCalc automatically looks up MYFUNC in FunctionList
  - Registration format: `SocialCalc.Formula.FunctionList['MYFUNC'] = [jsFunction, argCount, 'custom', 'Description', 'all']`
  - This is a room-scoped registry - each room's VM context has its own SocialCalc instance and FunctionList

- **Macros → `room.macros`** (or similar room-scoped Map):
  - Macros are NOT in FunctionList because they cannot be called from formulas
  - They are triggered explicitly via UI actions or API calls using a specific macroId
  - Registration format: `room.macros.set('myMacro', macroFunc)` or `room.macros['myMacro'] = macroFunc`
  - This is room-specific storage, allowing different rooms to have different macros with the same names
  - Lookup is direct by macroId, not through formula parsing

This separation ensures that macros don't pollute the formula namespace and that formulas can't accidentally invoke action-oriented macros that weren't designed to return cell values.

### Step 3: Execution

#### Function Execution (UDFs)

SocialCalc routes functions in formulas to their underlying JavaScript implementations via its FunctionList registry (a global object that maps function names to arrays containing the JS function and metadata).

Here's how it works based on the SocialCalc code in ethercalc.js:

- **Formula Parsing**: When evaluating a formula like `=MYFUNC(A1, B2)` for UDFs, SocialCalc's parser tokenizes it and identifies MYFUNC as a function call with arguments.

- **Lookup in FunctionList**: The evaluator calls `SocialCalc.Formula.CalculateFunction(funcName, operands, ...)`:
  - It checks `SocialCalc.Formula.FunctionList[funcName]` for a registered entry.
  - If found, the entry is an array like `[jsFunction, argCount, argType, description, category, ...]`.

- **Argument Evaluation and Execution**:
  - Arguments are evaluated (e.g., A1 resolves to its cell value).
  - The JS function is called: `jsFunction(funcName, operands, argsArray, sheet, cellCoord)`.
  - The function returns a value, which SocialCalc uses in the formula result.

This allows seamless integration—built-in functions (like SUM) and custom UDFs are treated identically. The VM executes them in the same sandboxed context, with routing handled entirely by SocialCalc's formula engine. If a function isn't in FunctionList, it errors as "Unknown function".

#### Macro Execution

Macros are executed differently than UDFs. Instead of being invoked from cell references, they are invoked when a user clicks through the UI to run a specific macro or when the API is triggered.

Here's how it works:

- **Trigger Identification**: A macro execution begins from:
  - **UI Trigger**: User selects and runs a macro through the UI
  - **API Trigger**: External system calls `POST /room/:id/macros/:macroId/run` (see [05-backend-api.md](05-backend-api.md))

- **Macro Lookup**: The server identifies and retrieves the macro by its `macroId`:
  - The server checks the room's macro registry (`room.macros` - a Map or object separate from FunctionList)
  - The macroId is used as the lookup key (e.g., `room.macros.get('myMacro')` or `room.macros['myMacro']`)
  - If found and already loaded, the macro function is retrieved directly
  - If not found in the registry:
    - The server scans configured repos for a `.macro.js` file with a matching filename
    - If found, the macro is loaded (Steps 1 & 2 above) and registered in `room.macros`
    - If not found in any repo, returns a "Macro not found" error

- **VM Execution**: The macro code is executed in a sandboxed Node.js VM:
  - Policies control what's available in the VM scope (see [04-security-permissions.md](04-security-permissions.md))
  - The VM is configured with timeout limits, memory constraints, and restricted APIs
  - Sheet manipulation capabilities are exposed in the VM's global scope (implementation details TBD)
  - See [11-sandboxing.md](11-sandboxing.md) for VM configuration and restrictions

- **Result Handling**:
  - Sheet modifications are persisted and trigger change tracking
  - Changes are broadcast to all connected clients via the collaboration system
  - For API calls, the macro's return value (if any) is sent in the HTTP response
  - Errors are caught, logged, and reported appropriately

**Real-time Updates**: When macros modify worksheet data, those changes are broadcast to all connected clients in real-time through EtherCalc's collaboration system. This ensures that users viewing the worksheet see macro-triggered updates immediately without needing to refresh. See [07-backend-api-security.md](07-backend-api-security.md#integration-with-broadcast-system) for details on broadcast system integration.

### Validation and Security

- **Code Validation**: Code is linted for syntax; policies restrict features (e.g., no 'require' for external modules). Invalid code is logged but skipped, preventing runtime errors.
- **Caching**: Loaded functions persist in the room's in-memory context (a JavaScript Map or object) for the session, shared across users in that room. Cache clears on room restart or config changes.

### UX Triggers for Loading
- When a formula calls a repo-qualified UDF (e.g., `myrepo.MYFUNC()`) for the first time in a session.
- When editing formulas in the formula bar, if autocomplete or validation requires checking available UDFs from repos.
- When opening the macro editor or repo configuration panel to display available macros from repos.
- When selecting a macro from a repo for editing or inspection.
- Optionally, when a worksheet with configured repos is first opened (configurable per deployment for performance vs. UX).
- When applying policies that require scanning repo contents (e.g., during room access or macro execution).

## Stateful Executions

It's possible to persist state in memory between runs within a room's execution context, as functions are registered in a shared JavaScript object/Map that persists for the session. UDFs (called in formulas) and macros (executed via API) can modify global vars, closures, or shared objects in that context (e.g., a counter or cache). However, strict sandboxing (via Node.js VM) prevents access to server globals or external resources, and the context resets on room restart or config changes. This allows controlled statefulness (e.g., for caching) but enforces isolation for security. If you need examples or to adjust this in the spec, let me know!

## Additional Security Considerations

- **Policy Application**: Repo-level policies (trust, sandboxing) applied to all macros from that repo.
- **Audit**: Repo access and credential usage logged without exposing secrets.
- **Access Control**: Repos are bound by the existing authentication model. e.g. if a user is an author they can edit the repo config or the repo, otherwise a security exception raises.

## Todos
- Call out RESTful endpoints for repo CRUD operations.
- **UI/UX Discussion and Diagram**: Discuss user interface design for repo management and create diagrams for user journeys.
  - **Relevant User Journeys**:
    - Adding a user-provided external VCS repo (user inputs URL and credentials).
    - Creating a server-provided external VCS repo (user requests creation, server handles setup).
    - Configuring multiple repos in a worksheet (adding/removing repos, setting defaults).
    - Executing functions from multiple repos (resolving conflicts, on-demand loading).
    - Sharing repos across worksheets (manually configuring the same repo in multiple worksheets).
- **Multi-Repo Function Resolution and Execution**: Define how functions from multiple repos are resolved, loaded, and executed, including conflict resolution and on-demand loading.
- **Credential Encryption Details**: Define the encryption mechanism for user-provided credentials in snapshots.
- **VCS Platform Modules**: Design the modular architecture for supporting different VCS platforms.
- **Policy Integration**: Detail how repo-level policies interact with global/room/macro policies.
- **API Design**: Specify RESTful endpoints for repo CRUD operations.
- **Testing Scenarios**: Outline test cases for multi-repo functionality and security.

**Topics from Recent Discussions (Not Yet Fully Incorporated):**

1. **MacroId/FunctionId Determination Rules**
   - Document that macroId comes from filename (calculateTax.macro.js → macroId "calculateTax"), not the JS function name
   - Same principle for UDF function names from .func.js files
   - Document what happens if filename is invalid or conflicts

2. **Fully-Qualified Registration**
   - Macros should ALWAYS register as "repoName.macroName" (e.g., "myrepo.calculateTax")
   - UDFs should ALWAYS register as "repoName.FUNCTIONNAME"
   - Update API endpoint format (`POST /room/:id/macros/:macroId/run`) - clarify if macroId is "myrepo.calculateTax" or just "calculateTax"
   - Update all registration examples to reflect hierarchical Map structure consistently

3. **Enhanced Non-Enumerable Properties**
   - Add `__ethercalc_repo`: repo name for sync tracking
   - Add `__ethercalc_filepath`: original filename for reload
   - Add `__ethercalc_hash`: content hash for change detection
   - Currently document only shows `__ethercalc_type` and `__ethercalc_id`

4. **Comprehensive Locking & Synchronization Mechanism**
   - Repo-level lock implementation (acquire/release, queue management)
   - In-flight execution tracking (activeExecutions Map with executionId, repoName, startTime)
   - Wait-for-executions logic with timeouts
   - Blocking new executions during updates (blockedRepos Set)
   - Decision on behavior: queue vs reject vs timeout

5. **Crash Recovery & Consistency**
   - Write-ahead logging for repo updates
   - Recovery procedures on server restart
   - Incomplete transaction handling
   - Version numbers for detecting mid-execution changes
   - Snapshot consistency (capturing registry version during save)

6. **UDF Registry Atomic Updates**
   - SocialCalc.Formula.FunctionList needs same CMS pattern as room.macros
   - UDFs are room-scoped (each room has its own FunctionList in its VM context)
   - How to handle formula evaluation during UDF repo updates within a specific room

7. **Update Trigger Scenarios**
   - Document all scenarios that trigger async updates (user-triggered, automated, multi-user, etc.)
   - Define WHEN syncs happen (manual, auto, periodic, webhook-triggered)
   - UI/UX for triggering sync operations

8. **Registry Structure Consistency**
   - Make hierarchical Map structure consistent throughout document
   - Update all code examples (Step 2 Registration, Macro Execution, etc.)
   - Remove references to flat structure

9. **Diff vs Wholesale Sync Decision**
   - Document which approach we're using
   - Address stateful macro preservation concerns
   - Tradeoffs between approaches

10. **Lazy Loading Interaction with Sync**
    - How to prevent stale code from loading after sync
    - Repo "dirty" marking strategy
    - Registry invalidation on sync

11. **Snapshot-Embedded vs External VCS Repo Differences**
    - Snapshot repos: sync on snapshot save (explicit user action)
    - External repos: sync on git pull/fetch (separate operation)
    - Different sync strategies for different repo types

12. **Multi-User Coordination**
    - What happens when User A syncs a repo while User B is executing a macro from it
    - Broadcast "repo updated" events to all connected clients
    - Client-side handling of registry changes

## References
- [03-storage-strategy.md](03-storage-strategy.md): Snapshot format and storage mechanisms for embedded repos.
- [04-security-permissions.md](04-security-permissions.md): Policy hierarchy and application to repos/macros.
- [05-backend-api.md](05-backend-api.md): API endpoints for repo management and operations.
- [05-backend-api.md](05-backend-api.md): "Type is determined by file extension (.func.js for UDFs, .macro.js for macros)." And "File Mapping: .macro.js files → macros; .func.js files → UDFs."

## UI Components
These represent self-contained UI modules for repo management, designed to be embedded in broader user journeys (see [06-ui-integration.md](06-ui-integration.md)). Each includes a spec and ASCII diagram.

### Component 1: Add User-Provided External VCS Repo
- **Description**: Modal dialog for users to connect their own VCS repo to a worksheet by entering URL and credentials.
- **Steps**:
  1. User clicks "Add Repo" > "External VCS" > "User-Provided".
  2. Form fields: Repo URL, Auth Type (Token/Password), Credentials input.
  3. Validation: Test connection before saving.
  4. Save: Encrypts credentials in snapshot.
- **ASCII Diagram**:
```
+-----------------------------+
| Add User-Provided VCS Repo  |
+-----------------------------+
| Repo Name: [______________]  |
| Repo URL: [______________]  |
| Auth Type: [Token v]        |
| Token: [______________]     |
| [Test Connection] [Cancel] [Add] |
+-----------------------------+
```

### Component 2: Create Server-Provided External VCS Repo
- **Description**: Dialog for requesting server to create a new VCS repo under server account and attach that repo to the worksheet.
- **Steps**:
  1. User clicks "Add Repo" > "External VCS" > "Server-Provided".
  2. Form fields: Repo Name (auto-suggested as roomId-based).
  3. Server API call to create repo on VCS.
  4. Confirmation: Repo URL displayed for sharing.
- **ASCII Diagram**:
```
+-----------------------------+
| Create Server VCS Repo      |
+-----------------------------+
| Repo Name: [room123-macros] |
| [Create] [Cancel]           |
|                             |
| Status: Creating...         |
+-----------------------------+
```

### Component 3: Configure Multiple Repos in Worksheet
- **Description**: Panel for managing repos attached to a worksheet (add/remove/set defaults). Note: if a server provided repo is removed, it is deleted. The embedded snapshot repo is named "default" and is default to every worksheet and cannot be deleted.
- **Steps**:
  1. Access via worksheet settings or macro editor.
  2. List of current repos with edit/remove buttons.
  3. "Add Repo" button launches sub-components.
  4. Set default repo for unqualified function calls.
- **ASCII Diagram**:
```
Worksheet Settings
+-----------------------------+
| Repos:                      |
| + default         [Edit]    |
| + myrepo          [Edit][X] |
| + sharedmacros    [Edit][X] |
|                             |
| [Add Repo]                  |
+-----------------------------+
```

## UI Modifications
These represent enhancements to existing UI functionality that are not self-contained in new modules.

### Modification 1: Fucntion resolution in cell reference
- **Description**: Enhancement to the existing function calling mechanism to support repo-qualified functions with intellisense and conflict resolution.
- **Steps**:
  1. In formula bar, typing shows suggestions with repo prefixes.
  2. Conflicts highlighted (e.g., "MYFUNC exists in default, myrepo").
  3. On-demand loading: First call to repo function triggers load.
- **ASCII Diagram**:
```
Formula: =default.MYFUNC(A1) + myrepo.OTHER(B2)
+-----------------------------+
| Suggestions:                |
| default.MYFUNC(...)         |
| myrepo.MYFUNC(...)  <conflict>|
| myrepo.OTHER(...)           |
+-----------------------------+
```

## Handling Asynchronous Repo Updates

We want the updates to be synchronous and atomic. We will use write locks and a Copy-Modify-Swap (CMS) Pattern. In Node.js, reference assignment is atomic. We willl leverage this as shown below:

```
// Current state
// Hierarchical structure: repo → macro name → function
room.macros = Map {
  'default': Map {
    'calc': func1,
    'summary': func2
  },
  'myrepo': Map {
    'tax': func3,
    'format': func4,
    'validate': func5
  },
  'other': Map {
    'util': func6
  }
}

// Load new repo version
newRepo = new Map(...)

// Update the macros map
newMacros.set('myrepo', newRepoMacros);
```

