## Table of Contents

- [Underlying Technology](#underlying-technology)
- [Ethercalc's current bootSC mechanism](#ethercalcs-current-bootsc-mechanism)
- [Proposed changes](#proposed-changes)
  - [Step 1: SocialCalc changes (core engine)](#step-1-socialcalc-changes-core-engine)
    - [New FunctionList Structure](#new-functionlist-structure)
    - [Function lookup changes](#function-lookup-changes)
    - [Formula parser updates](#formula-parser-updates)
    - [Error handling behavior](#error-handling-behavior)
  - [Step 2: EtherCalc integration (room lifecycle)](#step-2-ethercalc-integration-room-lifecycle)
    - [Post-bootSC restructure shim](#post-bootsc-restructure-shim)
    - [Repo loading logic](#repo-loading-logic)
    - [Persistence integration note](#persistence-integration-note)
  - [Step 3: UI autocomplete updates](#step-3-ui-autocomplete-updates)
    - [Current autocomplete functionality](#current-autocomplete-functionality)
    - [Changes needed for hierarchical structure](#changes-needed-for-hierarchical-structure)

## Underlying Technology

EtherCalc
- Repository: https://github.com/audreyt/ethercalc
- Nature: Node.js collaborative spreadsheet server application
- Purpose: Provides multi-user editing, persistence, web interface, and room-based isolation
- Dependencies: Includes SocialCalc as an npm package ("socialcalc": "^2.3.0")

SocialCalc
- Official Repository: https://github.com/DanBricklin/socialcalc
- Popular Fork (used by EtherCalc): https://github.com/marcelklehr/socialcalc
- Nature: Standalone JavaScript spreadsheet engine library
- Purpose: Core spreadsheet functionality (formula evaluation, cell management, parsing)
- Distribution: Published as npm package that EtherCalc loads via its bootSC mechanism

## Ethercalc's current bootSC mechanism

The bootSC object is a dynamically constructed JavaScript string that contains the entire SocialCalc library plus Node.js compatibility modifications. This string is ultimately evaluated by the VM created for the room (worksheet). The SocialCalc code contained in the string is what populates the FunctionList object used by SocialCalc to resolve function names etc.

1. **Load raw source code**
bootSC starts as the raw SocialCalc.js file content. The source code is literally read from the file included in the npm package.

    ```
    // Read the entire SocialCalc library as a string
    bootSC = fs.readFileSync(FindBin + "/node_modules/socialcalc/SocialCalc.js", 'utf8');
    ```

2. **Modify source for browser compatibility**
Ethercalc's codebase then modifies the SocialCalc code base to make it compatible with a browser 

    ```
    bootSC.=replace(/document\.createElement\(/g, 'SocialCalc.document.createElement(')
    bootSC.=replace(/alert\(/g, '(function(){})(')
    ```
    and node.js.
    ```
    bootSC += """;var navigator = {language: '', userAgent: ''}; var SocialCalc = this.SocialCalc; var window = this;(#{->
    // ... DOM shim code for Node.js environment
    })();"""
    ```
3. **Create VM Context**
Each EtherCalc room gets its own worker which in tern sets up an isolated JavaScript context using Node.js vm.createContext():

4. **Evaluate the bootSC string**
The bootSC string is then evaluated within the isolated VM context for the room.
    ```
    w.thread.eval bootSC, ~> w.postMessage { type: \init, room, log, snapshot }
    ```

5. **Set the function list**
The SocialCalc codebase being evaluated sets up the function list.

    ```
    // From within SocialCalc.js (executed during bootSC evaluation):
    SocialCalc.Formula.FunctionList = SocialCalc.Formula.FunctionList || {};

    // Built-in functions get registered:
    SocialCalc.Formula.FunctionList.SUM = [SocialCalc.Formula.SeriesFunctions, -1, "vn", null, "stat"];
    SocialCalc.Formula.FunctionList.AVERAGE = [SocialCalc.Formula.SeriesFunctions, -1, "vn", null, "stat"];
    SocialCalc.Formula.FunctionList.COUNT = [SocialCalc.Formula.SeriesFunctions, -1, "vn", null, "stat"];
    // ... 100+ more built-in functions
    ```

## Proposed changes

### New FunctionList Structure

### Step 1: SocialCalc changes (core engine)

#### New FunctionList Structure

We want the function list to change to a function map which mirrors the structure of the macros map. All of the builtin functions should be registered to the _builtin_ repo while the other functions are registered to their respective repo as they are written / discovered.

```
room.functionList = Map {
  '_builtin_': Map {
    'MYFUNC': myUdfFunc,
    'CALCTAX': calcTaxFunc
    //... Remaining 100+ built in fucntions from social calc
  },
  'default': Map {
    // ... UDFs stored directly in the snapshot
  },
  'myrepo': Map {
    'SPECIALFUNC': specialFunc
    // ... UDFs stored in the remote repo
  }
  }
}
```

#### Function lookup changes

The SocialCalc engine needs to resolve function names from the hierarchical Map structure. The following exact functions must be updated:
To support the hierarchical functionList structure (replacing the flat `SocialCalc.Formula.FunctionList` object with a Map of Maps for repo-organized UDFs), the following exact functions in SocialCalc need to be modified. These are based on the current usage patterns in the codebase (derived from the minified `static/ethercalc.js` and SocialCalc's formula evaluation logic):

1. **`SocialCalc.Formula.CalculateFunction`**  
   - **Current behavior**: Directly accesses `u.FunctionList[e]` where `e` is the function name (e.g., "SUM").  
   - **Required change**: Replace the direct lookup with a hierarchical lookup function that:  
     - Splits `e` on "." (e.g., "myrepo.SUM" → ["myrepo", "SUM"]).  
     - If qualified (has "."), looks up in `u.FunctionList.get(repo).get(func)`.  
     - If unqualified, searches fallback repos (e.g., "default", "_builtin") in order.  
     - Returns the function definition array or `null` if not found.  
   - **Why**: This is the core function evaluation entry point; all formula calculations go through here.

2. **`SocialCalc.Formula.FillFunctionInfo`**  
   - **Current behavior**: Iterates `for(e in a.FunctionList)` to build `FunctionClasses` and `FunctionArgDefs`. Also assigns descriptions back with `a.FunctionList[e][3] = ...`.  
   - **Required change**: Update to iterate over all repos in the hierarchical Map (e.g., `for (let [repoName, repoMap] of a.FunctionList)` then `for (let [funcName, funcDef] of repoMap)`), collecting functions from all repos while preserving class categorization. Change the assignment to modify the `funcDef` array directly (e.g., `funcDef[3] = ...`) instead of accessing `a.FunctionList[e][3]`.  
   - **Why**: Ensures function info/metadata (args, descriptions, classes) is populated from UDF repos, not just built-ins. The direct array modification avoids issues with Map access.

3. **`SocialCalc.Formula.FunctionArgString`**  
   - **Current behavior**: Directly accesses `a.FunctionList[e]` to get function args.  
   - **Required change**: Use the same hierarchical lookup function as `CalculateFunction` to retrieve the function definition, then extract args from it.  
   - **Why**: Used for displaying function signatures in the UI (e.g., tooltips during formula input).

4. **`SocialCalc.Formula.SetInputEchoText`**  
   - **Current behavior**: Checks `t.Formula.FunctionList[i]` for existence to show function prompts.  
   - **Required change**: Replace with a call to the hierarchical lookup function to check if the function exists (qualified or unqualified).  
   - **Why**: Provides autocomplete/intellisense for UDFs in the formula input echo.

5. **Unnamed input widget rendering function** (appears in the minified code around line 5 of `static/ethercalc.js`)  
   - **Current behavior**: Accesses `t.Formula.FunctionList[d]` for widget parameter handling.  
   - **Required change**: Use the hierarchical lookup function instead of direct access.  
   - **Why**: Ensures UDFs work in input widgets/forms.

6. **Implementation of Hierarchical Lookup Function**  
   - **Current behavior**: N/A (new function).  
   - **Required change**: Add a new function `SocialCalc.Formula.LookupFunction(funcName)` in SocialCalc that:  
     - Splits `funcName` on "." to get `repo` and `baseName`.  
     - If qualified: Returns `this.FunctionList.get(repo)?.get(baseName)` or `null`.  
     - If unqualified: Searches repos in order: `this.FunctionList.get('default')?.get(funcName)` then `this.FunctionList.get('_builtin')?.get(funcName)`.  
     - Handles case-insensitive matching if needed (SocialCalc functions are uppercase).  
   - **Why**: Provides a centralized utility for hierarchical function resolution, used by all the modified functions above to replace direct `FunctionList[funcName]` accesses.

#### Formula parser updates

**What’s missing:**
- The input echo regex allows dots, but the **core formula parser** must also treat `repo.func` as a single function identifier in all parsing paths.
- There’s no explicit normalization rule for qualified names (case handling for repo and function segments).
- Error messages don’t yet differentiate “unknown repo” vs “unknown function”.

**What to add:**
1. **Qualified name normalization utility** used by both parser and lookup (e.g., `NormalizeFunctionName(raw)` → `{ repo?, func }`).
2. **Parser validation** that treats `repo.func(` as a valid function token (not as a property access or name with dot).
3. **Parsing tests** for:
  - `=myrepo.SUM(1,2)`
  - `=MYREPO.sum(1,2)`
  - `=sum(1,2)` when only `_builtin_` exists
4. **Malformed qualifier handling** (e.g., `A..B`, `A.`) should produce a specific parse error.

#### Error handling behavior

**What is an “error type” vs a “message”?**
- **Error type** is the spreadsheet error token stored in the cell value (e.g., `#VALUE!`, `#NAME?`, `#DIV/0!`). This is what users see in the cell.
- **Message** is the human-readable explanation string passed alongside the error token (e.g., “Unknown function: SUM”). It is typically shown in the status area, formula bar, or in tooltips depending on the UI path.

**How users experience errors today:**
- If a formula fails, the cell displays the error token (e.g., `#NAME?`).
- The UI can surface a more descriptive message during editing or in the status/prompt output, depending on the interaction (e.g., input echo prompt).

**Recommendation:**
1. **Reuse existing error types** (keep `#NAME?`, `#VALUE!`, etc.) to preserve compatibility with SocialCalc’s display logic.
2. **Enhance error messages for qualified names** without inventing new error types:
  - Unknown repo: “Unknown repo: myrepo”.
  - Unknown function in repo: “Unknown function: myrepo.SUM”.
  - Unqualified ambiguity: “Unknown function: SUM (try myrepo.SUM)”.
3. **Route messages through existing error hooks** (e.g., `FunctionArgsError`, `FunctionSpecificError`, or the unknown-function path) so the UI surfaces the message without changing how error tokens render in cells.

### Step 2: EtherCalc integration (room lifecycle)

#### Post-bootSC Restructure Shim

We will need to add a shim that runs after the initial function list is created to transform it into the new map structure.

#### Repo Loading Logic

**Where to add this in the workflow:**
- Add a single, explicit hook immediately after the VM evaluates `bootSC` and after the function list is restructured into the hierarchical Map. This is the same place the snapshot is applied to the room.

**Concrete integration points:**
- **Worker init path** (room boot):
  1. Evaluate `bootSC`.
  2. Restructure `SocialCalc.Formula.FunctionList` into `Map` with `_builtin_` and empty `default`.
  3. Apply snapshot (sheet state).
  4. Populate UDF repos into `FunctionList` (default + external repos).

**Concrete population order (deterministic):**
1. `_builtin_`: move built-in functions from original flat list.
2. `default`: populate from snapshot UDFs.
3. External repos: for each repo in load order, attach a Map under its repo name.

**Update semantics:**
- When a repo refreshes, replace only that repo Map (copy‑modify‑swap) to keep updates atomic.
- When a repo unloads, remove only that repo entry.

**Conflict policy:**
- Qualified names always resolve to their repo.
- Unqualified names search `default`, then `_builtin_`.

**Implementation details:**
- **Hook location:** In `src/main.ls` (or equivalent), after the `w.thread.eval bootSC` call in the worker initialization. Add a new function `populateFunctionRepos(room, snapshot)` that runs post-bootSC.
- **Restructuring code:** Create a new Map for `SocialCalc.Formula.FunctionList`. Iterate the original flat object, moving all entries to `_builtin_` sub-Map. Initialize empty `default` sub-Map.
- **Snapshot UDF loading:** Parse the snapshot for UDF definitions (assuming they are stored in a specific format, e.g., as JSON objects with name and code). For each UDF, add to `default` sub-Map using `SocialCalc.Formula.FunctionList.get('default').set(udfName, udfDef)`.
- **External repo loading:** For each configured repo (from room config or global settings), fetch UDFs via HTTP/HTTPS or local file system. Parse and add to respective sub-Maps. Use async/await for network calls to avoid blocking room init.
- **Error handling:** If a repo fails to load, log the error but continue with other repos. Ensure the room can still function with partial UDF availability.
- **Testing:** Add unit tests for repo loading order, conflict resolution, and failure scenarios. Mock external repos for integration tests.

#### Persistence integration note

- Ensure UDFs in the `default` repo are serialized into snapshots and restored on room load.
- This is tracked as a TODO in the storage strategy document (MACRO-PLAN/03-storage-strategy.md).

**Implementation details:**
- **Serialization:** During snapshot creation (in `src/db.ls` or equivalent), iterate `SocialCalc.Formula.FunctionList.get('default')` and serialize each UDF as a JSON object with fields: `name`, `args`, `description`, `code` (the function definition array). Store as an array under a new snapshot key like `udfs`.
- **Storage format:** Example snapshot addition: `{ ..., "udfs": [{"name": "MYFUNC", "args": "vn", "description": "Custom function", "code": [functionRef, -1, "vn", null, "custom"]}] }`.
- **Restoration:** On room load, after restructuring the FunctionList, parse the `udfs` array from the snapshot and populate the `default` sub-Map using `SocialCalc.Formula.FunctionList.get('default').set(udf.name, udf.code)`.
- **Versioning:** Add snapshot version checks to handle UDF format changes. If no `udfs` key exists, initialize empty `default` Map.
- **Security:** Validate UDF code during restoration to prevent injection attacks. Use VM sandboxing if UDFs contain executable code.
- **Backup:** Ensure UDFs are included in room backups and exports. Test restoration from backups to verify UDF persistence.

### Step 3: UI autocomplete updates

Based on the code analysis, here's how the autocomplete/intellisense works in EtherCalc today and what needs to change for the hierarchical functionList:

#### Current autocomplete functionality

**Yes, autocomplete exists** in two forms:

##### 1. **Inline Formula Tooltips** (`SetInputEchoText`)
- **How it works**: As you type formulas, it uses a regex to detect function names: `/.*[\+\-\*\/\&\^\<\>\=\,\(]([A-Za-z][A-Za-z][\w\.]*?)\([^\)]*$/`
- This matches function calls like `=SUM(` and extracts the function name (allowing dots in names, which is good for qualified names).
- If `t.Formula.FunctionList[i]` exists (where `i` is the uppercase function name), it calls `FillFunctionInfo()` and shows a tooltip with `i + "(" + FunctionArgString(i) + ")"`.
- If not found, shows "Unknown function: i".

##### 2. **Function Insert Dialog** (`SpreadsheetControl.DoFunctionList`)
- **How it works**: A popup dialog (accessible via UI button) with:
  - **Category dropdown**: Lists function categories (stat, math, text, etc.) from `function_classlist` constants.
  - **Function dropdown**: Populated from `FunctionClasses[category].items` (built by `FillFunctionInfo`).
  - **Description panel**: Shows function signature and description using `FunctionArgString` and the function's `[3]` description field.
  - **Paste button**: Inserts the selected function into the current formula input.
- `FillFunctionInfo` categorizes all functions from `FunctionList` into classes like "stat" (SUM, AVERAGE), "math" (SIN, COS), etc.

#### Changes needed for hierarchical structure

##### 1. **Update FillFunctionInfo for Multi-Repo Categorization**
- **Current**: `for(e in a.FunctionList)` iterates flat object.
- **New**: Change to `for (let [repoName, repoMap] of a.FunctionList) { for (let [funcName, funcDef] of repoMap) { ... } }`
- **Impact**: Functions from all repos (_builtin_, default, myrepo) will be included in categories. For qualified names, consider adding repo prefix to category items (e.g., "myrepo.SUM" in "stat").

##### 2. **Modify Inline Tooltips for Qualified Names**
- **Current**: Regex already allows dots, lookup checks `FunctionList[i]`.
- **New**: Update the lookup to use the new `LookupFunction(i)` instead of direct `FunctionList[i]` access.
- **Impact**: Tooltips will work for both `SUM(` and `myrepo.SUM(`. For qualified names, show the full qualified signature.

##### 3. **Enhance Function Dialog for Repo Organization**
- **Add repo grouping**: Modify the category dropdown to include repo-based categories (e.g., "Built-in Functions", "Default UDFs", "myrepo Functions").
- **Show qualified names**: Update `GetFunctionNamesStr` and `FillFunctionNames` to include repo prefixes where appropriate (e.g., display "myrepo.SUM" instead of just "SUM" if ambiguous).
- **Update descriptions**: Ensure `GetFunctionInfoStr` works with qualified lookups.

##### 4. **UI Configuration Changes**
- **Repo awareness**: The function insert dialog (SpreadsheetControl.DoFunctionList) should be aware of loaded repos and dynamically populate the category dropdown based on available repos. Modify the dialog initialization to query `SocialCalc.Formula.FunctionList` keys and add categories like "Built-in Functions" (for _builtin_), "Default UDFs" (for default), and "myrepo Functions" (for each external repo).
- **Search/filtering**: Add a new search input field to the dialog above the category dropdown. Implement filtering logic in `DoFunctionList` to hide/show functions in the function dropdown based on the search term, matching against qualified names (e.g., typing "sum" shows "SUM", "myrepo.SUM", etc.).
- **Visual indicators**: In the function dropdown, prefix function names with repo indicators (e.g., "[builtin] SUM", "[default] MYFUNC", "[myrepo] SPECIALFUNC"). Alternatively, use CSS classes or icons next to items to distinguish built-in vs UDF functions. Update the dropdown population code in `FillFunctionNames` to include these prefixes.

##### 5. **Error Handling**
- **Inline tooltips (SetInputEchoText)**: When a function name is not found, instead of just "Unknown function: SUM", check if qualified versions exist (e.g., search all repos for functions named "SUM"). If found, suggest them in the message (e.g., "Unknown function: SUM. Did you mean myrepo.SUM or default.SUM?"). Modify the lookup logic in SetInputEchoText to perform this check using the hierarchical FunctionList.
- **Function insert dialog**: If a user types an unknown function in the search/filter field or selects a category with no matches, display similar suggestions in the description panel or as a dialog alert. Update DoFunctionList to include suggestion logic when populating or filtering functions.
- **Formula parsing errors**: For parse-time errors (e.g., invalid qualified names like "A..B"), enhance error messages in the formula bar or status area to specify the issue (e.g., "Invalid function qualifier: A..B"). This requires updates to SocialCalc's formula parser error reporting to distinguish qualifier errors from other parse errors.

The core infrastructure is there, but the UI needs updates to leverage the hierarchical lookup and display qualified function names appropriately. The regex already supports dots, so parsing qualified names isn't an issue.



