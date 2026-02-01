# UI Integration

## UI Components (Planned)
### Menu Items / Entry Points
- **Macros** (menu item): opens the Macros workspace (Excel analogy: Macros dialog).

### Sections / Panels
- **Macro list panel**: list/search macros with type, source (embedded/repo), and status.
- **Macro settings panel**: permissions, network access, policy summary, and run limits.
- **Repo link panel**: repo URL, branch selector, pinned commit, status.
- **Audit panel**: macro run history and errors (if enabled).
- **Results/Errors panel**: shows the last run output and errors for the selected macro, with a collapsible stack trace when available.

### Forms / Modals
- **UDF editor form**: name, args, JS body, validation.
- **Macro editor form**: name, args, JS body or command list.
- **Run macro dialog**: select macro, provide parameters, validate inputs on submit, show output/errors in the Results/Errors panel while the sheet stays editable.
- **Repo sync modal**: pull/commit/push/cherry-pick confirmations and results.

## User Journey (Excel-Like Flow)
1. **Open Macros** (menu) → **Macros workspace** appears.
2. **Select or create macro** → **UDF/Macro editor** opens.
3. **Save** → macro appears in **Macro list panel**.
4. **Run macro** (Run button in Macro list or editor toolbar) → **Run macro dialog** → execution results shown.
5. **Link repo (optional)** → **Repo link panel** → branch selection → **Repo sync modal**.
6. **Review audit** (optional) → **Audit panel**.

## User Journey Flow Diagram

```
[Menu: Macros]
        |
        v
[Macros Workspace]______________________________________________________
  |                                     |                              |
  v                                     v                              v
[Macro List]                      [Macro Settings]                   [Repo Link]
  |                                     |
  v                                     v
[UDF/Macro Editor]        [Results/Errors Panel]
  |
  v
[Run Macro Dialog] ---> [Results/Errors Panel]


[Repo Link] ---> [Select Branch] ---> [Repo Sync Modal] ---> [Updated Code]
```

Note: This should feel like Excel’s Macros dialog + run flow, with a separate area for repository linking.

## Macro Settings (Details)
- **Scope**: global / room / macro overrides (read-only where not editable).
- **Capabilities**: network access toggle (if allowed), execution time limit, step limit.
- **Permissions**: who can edit/run (admin-only, owners, collaborators).
- **Repo linkage**: show associated `repoId` and pinned commit if sourced from repo.
- **Audit**: last run status, last editor, last sync.

## Formula UX (Details)
- Show UDF names in the formula input area (cell edit mode and the formula bar).
- Include short descriptions and argument hints as the user types.

## Editing Concurrency (Details)
- Lock a macro during edit; other users see who is editing and can request unlock if authorized.
- Requires backend lock endpoints and storage (MACRO-PLAN/05-backend-api.md).

## Guided Git UI (Planned)
- Simple flow with buttons: Connect Repo, Select Branch, Pull, Commit, Push, Cherry-pick.
- Branch selection required; diffing and advanced workflows handled by external VCS hosts (e.g., GitHub).
- No embedded terminal in the initial version.

## Files Likely Touched
- Client bundle (e.g., `static/ethercalc.js` or build pipeline).
- SocialCalc tab definitions.
