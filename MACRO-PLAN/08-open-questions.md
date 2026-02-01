# Open Questions

1. **Where macros live**: Redis-only first, or embed in snapshot (Excel analogy: macros stored inside the workbook file)?
2. **Macro language**: JS-only, DSL-only, or hybrid (Excel analogy: VBA vs recorded macro steps)?
3. **Permissions**: Who can create/edit/run macros by default?
4. **Overrides**: Can admins grant user-level overrides beyond room/global limits? Scope per room, per macro, or user-wide?
5. **Compatibility**: Must the macro system keep full backward compatibility with existing SocialCalc save files?

## Remaining todos for planning
- Repo sync: without a manifest or stable mapping, `.macro.js` discovery can drift or be ambiguous.
