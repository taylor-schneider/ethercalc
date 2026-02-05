# Backend API Security

## Table of Contents
- [Overview](#overview)
- [Authentication and Authorization](#authentication-and-authorization)
- [Policy Enforcement](#policy-enforcement)
- [Locking Mechanism](#locking-mechanism)
- [Endpoint Security Checks](#endpoint-security-checks)
  - [Macros Endpoints](#macros-endpoints)
  - [Macro Repos Endpoints](#macro-repos-endpoints)
  - [Locks Endpoints](#locks-endpoints)
  - [Policies Endpoints](#policies-endpoints)
- [Error Handling](#error-handling)
- [Audit Logging](#audit-logging)
- [Dependencies](#dependencies)

## Overview
This document outlines the security checks required for the backend API endpoints defined in [05-backend-api.md](05-backend-api.md). It integrates with the permission model in [04-security-permissions.md](04-security-permissions.md), ensuring macro operations respect EtherCalc's Editor/Viewer/Admin roles, hierarchical policies, and sandbox constraints.

Key principles:
- All endpoints require authentication via auth tokens (Editor: `auth=<hmac(room)>`, Viewer: `auth=0`).
- Policy checks enforce the hierarchy (global → room → macro → user overrides).
- Editing operations require locks to prevent concurrency.
- Admin operations are restricted and audited.

## Authentication and Authorization
- **Token Validation**: Extract `auth` from headers/query params. Validate against room HMAC for Editors.
- **Roles**:
  - **Editor**: Can create/edit/run macros; access most endpoints.
  - **Viewer**: Read-only; limited to non-modifying endpoints if allowed.
  - **Admin**: Can modify global/room policies; requires additional admin token/role check.
- **Room Scoping**: All room-related endpoints validate the user has access to the room.

## Policy Enforcement
- **Hierarchy Check**: Before operations, query and merge policies. Reject if operation violates (e.g., network disabled but macro requires it).
- **AST Validation**: For code uploads/syncs, parse and validate against policies (single function, whitelisted imports).
- **Runtime Checks**: During execution, enforce limits in the sandbox.
- **Override Handling**: User overrides can loosen restrictions only if server permits; audited.

## Locking Mechanism
- **Scope**: Per `roomId + macroId` (or repo).
- **TTL**: 10-30 minutes, refreshable.
- **Checks**: Acquire before edits; release after. Shared for code and policy changes.

## Endpoint Security Checks

### Macros Endpoints
- **`GET /room/:id/macros`** (list macros)
  - Auth: Editor only.
  - Policy: None.
  - Locks: None.
  - Rationale: Prevents Viewers from accessing macro code.

- **`PUT /room/:id/macros`** (update macros)
  - Auth: Editor only.
  - Policy: Validate code/policies; AST check.
  - Locks: Acquire room/macro lock.
  - Rationale: Ensures safe, atomic updates with policy compliance.

- **`POST /room/:id/macros/:macroId/run`** (execute macro)
  - Auth: Editor only.
  - Policy: Check runtime limits/network.
  - Locks: None.
  - Rationale: Execution must respect policies to prevent abuse.

### Macro Repos Endpoints
- **`GET /room/:id/macros/repos`** (list repos)
  - Auth: Editor only.
  - Policy: None.
  - Locks: None.

- **`PUT /room/:id/macros/repos/:repoId`** (update repo)
  - Auth: Editor only.
  - Policy: Validate URL against network policies.
  - Locks: Acquire repo lock.

- **`GET /room/:id/macros/repos/:repoId`** (fetch metadata)
  - Auth: Editor only.
  - Policy: None.
  - Locks: None.

- **`POST /room/:id/macros/repos/:repoId/sync`** (sync repo)
  - Auth: Editor only.
  - Policy: Validate synced code.
  - Locks: Acquire during sync.

### Locks Endpoints
- **`POST /room/:id/macros/:macroId/lock`** (acquire)
  - Auth: Editor only.
  - Policy: None.
  - Locks: Attempt acquire.

- **`DELETE /room/:id/macros/:macroId/lock`** (release)
  - Auth: Editor only (owner).
  - Policy: None.
  - Locks: Release if owned.

- **`GET /room/:id/macros/:macroId/lock`** (status)
  - Auth: Editor only.
  - Policy: None.
  - Locks: None.

### Policies Endpoints
- **`GET /policies/global`**
  - Auth: Any (or restricted).
  - Policy: None.

- **`PUT /policies/global`**
  - Auth: Admin only.
  - Policy: Audit; may require restart.

- **`GET /policies/user/:userId`**
  - Auth: User or Admin.

- **`PUT /policies/user/:userId`**
  - Auth: Admin only.
  - Policy: Audit overrides.

- **`DELETE /policies/user/:userId`**
  - Auth: Admin only.
  - Policy: Audit.

- **`GET /policies/room/:roomId`**
  - Auth: Room Editors or Admin.

- **`PUT /policies/room/:roomId`**
  - Auth: Room Admin.
  - Policy: Tighten only; audit.

- **`GET /policies/macro/:macroId`**
  - Auth: Editor.

- **`PUT /policies/macro/:macroId`**
  - Auth: Editor.
  - Policy: Tighten only.
  - Locks: Same as macro edit.

## Error Handling
- `403 Forbidden`: Auth/policy violation (e.g., "Permission denied: network access not allowed").
- `423 Locked`: Resource locked.
- Other codes as in [05-backend-api.md](05-backend-api.md).

## Audit Logging
- Log policy changes, macro executions, and admin actions to DB (e.g., "audit-" + room).

## Dependencies
- [04-security-permissions.md](04-security-permissions.md) for model details.
- [05-backend-api.md](05-backend-api.md) for endpoint specs.
- [11-sandboxing.md](11-sandboxing.md) for runtime enforcement.</content>
<parameter name="filePath">/root/ethercalc/MACRO-PLAN/07-backend-api-security.md