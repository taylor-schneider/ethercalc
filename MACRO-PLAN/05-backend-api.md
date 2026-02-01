# Backend API

## Overview
- UDFs are used via formulas; macros (action macros) are executed via the run endpoint.
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
- `POST /room/:id/macros/repos/:repoId/sync` — perform explicit repo sync (pull/checkout/cherry-pick). Only `.macro.js` files are loaded as macros.

### Locks
- `POST /room/:id/macros/:macroId/lock` — acquire lock.
- `DELETE /room/:id/macros/:macroId/lock` — release lock.
- `GET /room/:id/macros/:macroId/lock` — fetch lock status.
	- Locking is keyed by `roomId + macroId` (optionally `repoId`), TTL 10–30 minutes, refreshable.

## Object Model

### Macro
- `macroId`: stable identifier (normalized name, optional UUID for renames).
- `name`: display name.
- `type`: `udf` or `macro`.
- `args`: ordered list of argument names.
- `body`: JS source (or command list).
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

## Errors
- `400` Validation errors (name, args, payload size).
- `403` Permission denied (capability or policy conflict).
- `404` Macro not found.
- `408` Timeout (macro exceeded time budget).
- `409` Conflict or `423` Locked when a macro or repo is locked for editing.

## Dependencies
- UI additions described in MACRO-PLAN/06-ui-integration.md.
- Security policies described in MACRO-PLAN/04-security-permissions.md.
