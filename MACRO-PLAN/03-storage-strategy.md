# Storage Strategy (Planned Features)

## Table of Contents
- [Storage Strategy (Planned Features)](#storage-strategy-planned-features)
  - [Table of Contents](#table-of-contents)
  - [Macro Storage](#macro-storage)
    - [Feature 1: Workbook-Embedded Macros](#feature-1-workbook-embedded-macros)
    - [Feature 2: External Repository (Version-Pinned)](#feature-2-external-repository-version-pinned)
  - [Policy Storage](#policy-storage)
    - [Global Policy Storage](#global-policy-storage)
    - [Per-Room Policy Storage](#per-room-policy-storage)
    - [Per-Macro Flags Storage](#per-macro-flags-storage)
    - [Per-User Override Storage](#per-user-override-storage)
    - [Versioning and Compatibility](#versioning-and-compatibility)
    - [Dependencies](#dependencies)
  - [Remaining todos for planning](#remaining-todos-for-planning)

## Macro Storage

### Feature 1: Workbook-Embedded Macros
Store macro definitions inside the spreadsheet save snapshot.

### Feature 2: External Repository (Version-Pinned)
Store macro code in a git repository and reference it by version/tag/commit.

## Policy Storage
Policies are stored at different levels to match the hierarchy (global → room → macro → user overrides). This ensures policies persist and are enforceable across sessions. For consistency and atomic updates, room and macro policies are stored alongside the code/snapshots to prevent orphaned policies.

### Global Policy Storage
- **Location**: Database table (`global_policies`).
- **Content**: Default sandbox settings (e.g., network disabled, max runtime 500ms).
- **Updates**: Require server restart or admin API call; audited.
- **Benefits**: Centralized control; easy to backup/restore.
- **Trade-offs**: Changes affect all rooms; no per-room override without explicit config.

### Per-Room Policy Storage
- **Location**: Embedded in room snapshot (new section `--EtherCalc-Macro-Policies--`).
- **Content**: Room-specific restrictions (e.g., disable network for this room).
- **Updates**: Saved with room edits; versioned with snapshot.
- **Benefits**: Policies travel with the room; easy cloning.
- **Trade-offs**: Increases snapshot size; requires parsing on load.

### Per-Macro Flags Storage
- **Location**: Embedded with macro definition in snapshot (as metadata in macro JSON).
- **Content**: Macro-specific limits (e.g., `{"networkEnabled": false}`).
- **Updates**: Modified when editing macro; locked during edit.
- **Benefits**: Flags are part of the macro; consistent across rooms.
- **Trade-offs**: Can't override macro flags without editing the macro.

### Per-User Override Storage
- **Location**: User-specific DB table (`user_overrides`).
- **Content**: Trusted elevations (e.g., allow user X network access).
- **Updates**: Admin-set via API; audited with user/room/macro context.
- **Benefits**: Flexible for trusted users; doesn't affect snapshots.
- **Trade-offs**: Requires user identity tracking; potential for abuse if not audited.

### Versioning and Compatibility
- All policy storage includes a version field (e.g., `"policyVersion": "1.0"`).
- Unknown versions are ignored (fallback to defaults).
- Backward compatibility: older snapshots without policies use global defaults.

### Macro Storage Schema
Macros are stored in a new snapshot section `--EtherCalc-Macros--` with JSON format for easy parsing. The section includes a version header for compatibility.

**Snapshot Extension Format**:
```
--EtherCalc-Macros--
version: 1.0
macros: [
  {
    "id": "macro_123",
    "name": "MyMacro",
    "type": "macro",  // or "udf"
    "code": "function() { ... }",
    "args": ["arg1", "arg2"],  // for UDFs
    "policies": {
      "networkEnabled": false,
      "maxRuntimeMs": 1000,
      "maxSteps": 10000
    },
    "source": "embedded",  // or "repo"
    "repoId": null,  // if sourced from repo
    "commit": null,  // pinned commit hash
    "author": "user@example.com",
    "created": "2023-10-01T12:00:00Z",
    "modified": "2023-10-01T12:00:00Z"
  }
]
--End-EtherCalc-Macros--
```

**Parsing Rules**:
- If version is unknown (e.g., 2.0), skip the entire section (fallback to no macros).
- Macros with invalid JSON are skipped with warning.
- UDFs must have `args` array; macros can omit it.
- Policies default to global if missing.

**Benefits**: Extends existing snapshot format; versioned for future changes; embedded for portability.

### Dependencies
- Links to MACRO-PLAN/04-security-permissions.md for hierarchy details.
- Backend API endpoints for reading/writing policies (MACRO-PLAN/05-backend-api.md), including /policies/* endpoints for all policy levels.

## Remaining todos for planning
- Define a versioned snapshot extension for macros so older sheets still load. Proposed approach: add a clearly delimited macro section with a version header, and ignore it if unknown. **DONE**: Added `--EtherCalc-Macros--` section with version header.
- Backend data schema for macro storage inside snapshots (format + parsing) is not defined in MACRO-PLAN/03-storage-strategy.md. **DONE**: Defined JSON schema with fields for id, name, type, code, args, policies, source, repoId, commit, author, timestamps.
- Define a concrete macro storage schema with versioning and backward-compat behavior. **DONE**: Schema includes version field; unknown versions skipped.

