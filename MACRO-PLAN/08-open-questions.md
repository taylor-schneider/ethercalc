# Open Questions

## Table of Contents
- [Resolved](#resolved)
- [Still Open](#still-open)
- [Remaining todos for planning](#remaining-todos-for-planning)

## Resolved
1. **Where macros live**: Workbook-embedded macros are the primary source of truth, with optional external repo sync feeding updates into the embedded snapshot.
2. **Macro language**: JS-only (UDFs and macros in JavaScript), with sandboxing and policy controls.
3. **Permissions (default)**: Align macro permissions with the existing view/edit model. Users with edit auth (`auth=<hmac>`, i.e., edit mode) can create, edit, and run macros/UDFs by default. View-only users (`auth=0` or `view=1`) cannot create/edit/run macros. Per-room policy and per-macro flags may further restrict, and per-user overrides (if enabled) are the only way to exceed those defaults.

## Still Open
1. **Overrides**: Can admins grant user-level overrides beyond room/global limits? Scope per room, per macro, or user-wide?
2. **Compatibility**: Must the macro system keep full backward compatibility with existing SocialCalc save files?

## Remaining todos for planning
- Repo sync: define the `.macro.js` → `macroId` mapping rules to avoid ambiguity.
