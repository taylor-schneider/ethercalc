# Storage Strategy (Planned Features)

## Feature 1: Workbook-Embedded Macros
Store macro definitions inside the spreadsheet save snapshot.

Benefits:

Trade-offs:

Excel analogy:

## Feature 2: External Repository (Version-Pinned)
Store macro code in a git repository and reference it by version/tag/commit.

Benefits:

Trade-offs:

Notes:

## Dependencies

## Remaining todos for planning
- Define a versioned snapshot extension for macros so older sheets still load. Proposed approach: add a clearly delimited macro section with a version header, and ignore it if unknown.
- Backend data schema for macro storage inside snapshots (format + parsing) is not defined in MACRO-PLAN/03-storage-strategy.md.
- Define a concrete macro storage schema with versioning and backward-compat behavior.
