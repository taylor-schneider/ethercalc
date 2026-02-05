# Backend API

## Table of Contents
- [Overview](#overview)
- [Endpoints](#endpoints)
  - [Macros](#macros)
  - [Macro Repos](#macro-repos)
  - [Locks](#locks)
  - [Policies](#policies)
- [Object Model](#object-model)
  - [Macro](#macro)
  - [MacroRepo](#macro-repo)
  - [MacroLock](#macrolock)
  - [GlobalPolicy](#globalpolicy)
  - [UserPolicyOverride](#userpolicyoverride)
  - [RoomPolicy](#roompolicy)
  - [MacroPolicy](#macropolicy)
- [Error Handling](#error-handling)
- [Behavior Notes](#behavior-notes)

## Overview
- UDFs are used via formulas; macros (action macros) are executed via the run endpoint. Type is determined by file extension (`.func.js` for UDFs, `.macro.js` for macros).
- Repo sync endpoints are only used by the guided UI; execution uses embedded code.
- Editing uses a lock (mutex) so only one edit session is active at a time.
- All macro executions should be logged if audit is enabled.

## Endpoints

### Macros
- `GET /room/:id/macros` — list macros for a room (room = sheet/session).
- `PUT /room/:id/macros` — replace/update macros for a room.
- `POST /room/:id/macros/:macroId/run` — execute an action macro (Excel analogy: run a macro). Returns `404` if the macro does not exist.

### Macro Repos
- `GET /room/:id/macros/repos` — list macro repos linked to the room.
- `PUT /room/:id/macros/repos/:repoId` — set/update repo metadata (URL, branch or commit ref).
- `GET /room/:id/macros/repos/:repoId` — fetch repo metadata.
- `GET /room/:id/macros/repos/:repoId/tree` — fetch repo file tree (for file browser UI). Returns JSON tree structure with file types, paths, and last commit info for `.macro.js` and `.func.js` files.
- `POST /room/:id/macros/repos/:repoId/sync` — perform explicit repo sync (pull/checkout/cherry-pick). Only `.macro.js` and `.func.js` files are loaded as macros/UDFs.

**Repo Sync Flow Details**:
- **File Mapping**: `.macro.js` files → macros; `.func.js` files → UDFs. `macroId` derived from filename (e.g., `myMacro.macro.js` → `myMacro`).
- **Conflict Rules**: If `macroId` exists, update code if different; preserve policies unless overridden. New macros added; deleted files remove macros.
- **Commit Pinning**: Sync pins to specific commit; subsequent syncs check for changes.
- **Error Handling**: Git errors (e.g., invalid repo) return 400; file parse errors skip invalid files with warnings.

### Locks
- `POST /room/:id/macros/:macroId/lock` — acquire lock.
- `DELETE /room/:id/macros/:macroId/lock` — release lock.
- `GET /room/:id/macros/:macroId/lock` — fetch lock status.
	- Locking is keyed by `roomId + macroId` (optionally `repoId`), TTL 10–30 minutes, refreshable.

### Policies
- `GET /policies/global` — retrieve current global policies.
- `PUT /policies/global` — update global policies (audited; may require server restart for some changes).
- `GET /policies/user/:userId` — retrieve per-user policy overrides.
- `PUT /policies/user/:userId` — update per-user policy overrides (audited).
- `DELETE /policies/user/:userId` — remove per-user policy overrides (audited).
- `GET /policies/room/:roomId` — retrieve per-room policies.
- `PUT /policies/room/:roomId` — update per-room policies (saved with room edits).
- `GET /policies/macro/:macroId` — retrieve per-macro flags (scoped to room context if needed; uses same lock as macro editing).
- `PUT /policies/macro/:macroId` — update per-macro flags (modified during macro editing; requires same lock as macro).

## Object Model

### Macro
- `macroId`: stable identifier (normalized name, optional UUID for renames).
- `name`: display name.
- `type`: `udf` or `macro` (inferred from file extension: `.func.js` → `udf`, `.macro.js` → `macro`).
- `args`: ordered list of argument names.
- `body`: JS source (single function with optional top-level imports if policy allows).
- `source`: `embedded` or `repo`.
- `repoId` (optional): linked repo reference.
- Name collisions: if two macros normalize to the same `macroId`, reject with `409 Conflict` and require rename (or allow suffixing via UI, e.g., `name-2`).
- Duplicate definitions: the system will never silently overwrite an existing macro. Updates must target the exact `macroId` you intend to change.

### MacroRepo
- `repoId`: stable identifier (hash of `{repoUrl, path}` only).
- `repoUrl`: repository URL.
- `refType`: `branch` or `commit`.
- `ref`: branch name or commit SHA.
- `path` (optional): subdirectory path.
- `status`: last sync status.
- Repo updates: changing `refType` or `ref` updates the repo metadata without changing `repoId`.

### MacroLock
- `lockId`: unique lock identifier.
- `ownerId`: user id holding the lock.
- `ownerDisplayName`: display name for UI.
- `createdAt`: timestamp.
- `expiresAt`: timestamp.

### GlobalPolicy
- `policyVersion`: version string (e.g., "1.0").
- `settings`: object with sandbox settings (e.g., `{"networkEnabled": false, "maxRuntimeMs": 500}`).
- `updatedAt`: timestamp.
- `updatedBy`: admin user ID.

### UserPolicyOverride
- `userId`: user identifier.
- `policyVersion`: version string (e.g., "1.0").
- `overrides`: object with policy overrides (e.g., `{"networkEnabled": true}`).
- `updatedAt`: timestamp.
- `updatedBy`: admin user ID.

### RoomPolicy
- `roomId`: room identifier.
- `policyVersion`: version string (e.g., "1.0").
- `settings`: object with room-specific restrictions (e.g., `{"networkEnabled": false}`).
- `updatedAt`: timestamp.
- `updatedBy`: user ID.

### MacroPolicy
- `macroId`: macro identifier.
- `roomId`: room identifier (for scoping).
- `policyVersion`: version string (e.g., "1.0").
- `flags`: object with macro-specific limits (e.g., `{"networkEnabled": false}`).
- `updatedAt`: timestamp.
- `updatedBy`: user ID.

## Errors
- `400` Validation errors (name, args, payload size).
- `403` Permission denied (capability or policy conflict).
- `404` Macro not found.
- `408` Timeout (macro exceeded time budget).
- `409` Conflict or `423` Locked when a macro or repo is locked for editing.

## Dependencies
- UI additions described in MACRO-PLAN/06-ui-integration.md.
- Security policies described in MACRO-PLAN/04-security-permissions.md.

## Remaining todos for planning
- Repo sync flow details (how `.macro.js` files map to `macroId`, conflict rules) are still thin in MACRO-PLAN/05-backend-api.md. **DONE**: Added file mapping, conflict rules, commit pinning, error handling.
- Endpoints and objects might be implemented differently unless strictly aligned with existing server patterns. **NOTE**: Align with existing patterns (e.g., room-based auth like sheet commands).
- Specify `.macro.js` mapping rules (file naming → `macroId`, conflicts, updates). **DONE**.
- Repo file browser API for UI. **DONE**: Added GET /room/:id/macros/repos/:repoId/tree endpoint.
