# Storage Strategy (Planned Features)

## Feature 1: Workbook-Embedded Macros
Store macro definitions inside the spreadsheet save snapshot.

Benefits:
- Single artifact for portability (Excel analogy: macros embedded in the workbook file).
- Macros move with the sheet.

Trade-offs:
- Requires custom parsing and backward-compat handling.
- Larger snapshots and more complex merges.

Excel analogy:
- Macros are typically stored inside the workbook file as a VBA project (.xlsm). They can also be packaged as add-ins (.xlam).

## Feature 2: External Repository (Version-Pinned)
Store macro code in a git repository and reference it by version/tag/commit.

Benefits:
- Strong versioning and auditability.
- Easy rollback and CI validation.
- Encourages reusable macro libraries.

Trade-offs:
- Requires repository access and secret management.
- Adds dependency management and sync/update logic.
- Execution uses the embedded code; repo access is only needed during explicit sync actions.

Notes:
- Intended for trusted/owner mode or admin-approved rooms only.
- Optional caching can be used for repo sync operations (not required for macro execution).
- Guided UI will handle branch selection and basic sync (pull/commit/push); advanced git workflows remain in the VCS host UI.

## Dependencies
- UI additions described in MACRO-PLAN/06-ui-integration.md.
- Backend support for repository metadata storage per room (TBD: backend spec file).
- Backend support for git operations (pull/commit/push/cherry-pick) with permission checks (TBD: backend spec file).
