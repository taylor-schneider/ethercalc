# Storage Strategy (Planned Features)

## Table of Contents
- [Storage Strategy (Planned Features)](#storage-strategy-planned-features)
  - [Table of Contents](#table-of-contents)
  - [Background Info about Storage in EtherCalc](#background-info-about-storage-in-ethercalc)
  - [Macro Storage](#macro-storage)
    - [Feature 1: Workbook-Embedded Macros](#feature-1-workbook-embedded-macros)
    - [Feature 2: External Repository (Version-Pinned)](#feature-2-external-repository-version-pinned)
  - [Policy Storage](#policy-storage)
    - [Global Policy Storage](#global-policy-storage)
    - [Per-Room Policy Storage](#per-room-policy-storage)
    - [Per-Macro Flags Storage](#per-macro-flags-storage)
    - [Per-User Override Storage](#per-user-override-storage)
    - [Versioning and Compatibility](#versioning-and-compatibility)
    - [Macro Storage Schema](#macro-storage-schema)
    - [Dependencies](#dependencies)
  - [Remaining todos for planning](#remaining-todos-for-planning)

## Background Info about Storage in EtherCalc



**Spreadsheet Storage**: EtherCalc stores spreadsheets server-side only - there's no local storage for users. EtherCalc uses **snapshots** as the primary storage mechanism for spreadsheets. A snapshot is the serialized state of a spreadsheet in SocialCalc save format - a text-based format that contains all sheet data, formulas, formatting, and other spreadsheet state. Snapshots are stored in the database (Redis or on-disk JSON) with keys like `snapshot-{roomId}` and are loaded when a room (spreadsheet session) initializes.

Snapshots are persisted in the server's database and accessed via URLs like `/_/roomname`. Users don't download/upload spreadsheet files; all data stays server-side and is accessed through the web interface.

**Document Lifecycle**: Users create documents by accessing URLs like `/_/newroomname` or through API POST requests with spreadsheet data. The system generates unique room names (using UUIDs) for new documents. When users close their browser or session, the document data persists in the database. Users can return later by accessing the same URL - the system loads the snapshot from the database and reconstructs the spreadsheet state.

**Access Control and User Isolation**: Documents are accessed by URL (room name) with no built-in user ownership or private document lists. Anyone who knows or guesses a room name can access the document if they have the correct authentication token:
- `?auth=0` provides view-only access
- `?auth=<hmac(room)>` provides edit access (HMAC generated server-side using a secret key)
- Without proper authentication, API requests are rejected

For example, a spreadsheet named "budget2024" would be accessed at:
- View: `https://ethercalc.example.com/_/budget2024?auth=0`
- Edit: `https://ethercalc.example.com/_/budget2024?auth=<generated-hmac>`

**Critical Security Limitation**: EtherCalc's access control is fundamentally "security through obscurity" with significant implications for multi-user access and document security.

**How the /edit Endpoint Works**: The `/edit` endpoint (e.g., `/_/budget2024/edit`) is not a protected administrative function - it's a public utility that generates edit HMAC tokens for **any visitor** who knows the room name. When accessed, it simply redirects to the full edit URL with a freshly generated HMAC token. There is **no user authentication, authorization check, or rate limiting** on this endpoint.

**Multi-User Access Implications**: 
- **Anyone with the room name gets edit access**: If multiple people know or guess "budget2024", they can all obtain edit tokens via `/_/budget2024/edit`
- **No user isolation**: EtherCalc has no concept of document ownership or user-specific permissions. Documents are not "yours" - they're accessible to anyone with the URL
- **Shared editing by default**: All users with edit tokens can modify the same document simultaneously, with changes synced in real-time
- **No audit trail of who accessed what**: While edit operations are logged, there's no tracking of which specific users obtained edit tokens

**Security Through Obscurity Design**: EtherCalc relies entirely on room names being difficult to guess rather than proper access controls. The HMAC tokens provide:
- **Cryptographic integrity**: Tokens can't be forged without the server secret
- **No authorization**: Tokens don't restrict who can obtain them
- **No revocation**: Once obtained, tokens work until the server secret changes

**Practical Security Level**: Room names are the only protection. Short, predictable names like "budget", "test", "sheet1" are easily guessable. UUID-generated names provide better security, but users often choose memorable names, reducing security.

**Implications for Macro Security**: This access model has critical consequences for macro permissions:
- **No trusted user concept**: Since anyone can get edit access, macro policies can't rely on "trusted editors" vs "untrusted viewers"
- **Conservative security defaults required**: Macros must assume hostile editors and implement strict sandboxing
- **No per-user policy overrides**: The "per-user override" storage mentioned below is theoretical - EtherCalc has no user identity system to enforce it
- **Public macro execution**: Any macro in a publicly-named room can be executed by anyone who finds the room

**Real-World Usage Patterns**: Despite these limitations, EtherCalc works because:
- Many deployments use auto-generated UUID room names
- Documents are often shared intentionally via URL
- The real-time collaborative editing is the primary use case
- Security relies on communication channels outside EtherCalc (email, chat, etc.) to share room names

This security model prioritizes ease of collaboration over access control, making EtherCalc suitable for public wikis or intentionally shared documents, but unsuitable for private or sensitive data without additional security layers (VPN, Apache LDAP, etc.).

**Token Recovery and Orphaned Sheets**: If a user loses their HMAC token, they can regenerate it by visiting the `/edit` endpoint: `https://ethercalc.example.com/_/budget2024/edit` redirects to the full edit URL with a fresh HMAC token. However, if users forget the room name itself, sheets become effectively orphaned - there's no built-in admin panel or user account system to list "my documents." The `_rooms` API endpoint (disabled with CORS) can list all room names, but provides no ownership information. Sheets are only truly orphaned if both the room name AND the HMAC token are lost, with no server-side recovery mechanism.

This means EtherCalc documents are effectively "public by URL" - there's no user-specific document ownership or access control beyond URL-based authentication. Users can access other users' documents if they know the room names and have valid auth tokens.

Snapshots are created by SocialCalc's `CreateSpreadsheetSave()` method and parsed back using `DecodeSpreadsheetSave()`. This format is extensible, allowing new sections to be added for features like macros while maintaining backward compatibility with older snapshots that don't contain these sections.

**User Differentiation**: EtherCalc uses a simple, URL-based authentication system rather than user accounts. Users get random usernames for sessions, with view permissions via `?auth=0` and edit permissions via `?auth=<hmac>` (HMAC generated server-side). There's no user registration - users are differentiated by auth tokens in URLs.

**LDAP Integration**: EtherCalc supports LDAP authentication through Apache proxy configuration (`apache/apache-ldap.conf`), but full integration with EtherCalc's HMAC token system requires additional custom development. See `MACRO-PLAN/ldap-integration.md` for details.

## Macro Storage

### Feature 1: Workbook-Embedded Macros
Store macro definitions inside the spreadsheet save snapshot.

### Feature 2: External Repository (Version-Pinned)
Store macro code in a git repository and reference it by version/tag/commit. The repo can be either a locally stored git repository on the server or a connection to a remote git server using credentials (e.g., deploy keys, access tokens, or basic auth), depending on deployment needs.

**Credential Storage**: Remote repo credentials should be stored in server-side configuration only (never in snapshots). Recommended options include environment variables or a secrets manager/keystore (e.g., OS credential store, Vault, or Kubernetes secrets). Repo metadata stored in snapshots should only reference a credential ID or key name, not the secret itself.

**Credential Retrieval Mechanism**: 
- **Configuration Setup**: Credentials are configured server-side using environment variables prefixed by `ETHERCALC_GIT_CRED_` (e.g., `ETHERCALC_GIT_CRED_GITHUB_TOKEN=your_token_here`). Each credential gets a unique ID (e.g., `github_main`, `gitlab_private`).
- **Snapshot Reference**: Snapshots store only the credential ID (e.g., `"credentialId": "github_main"`) in the macro metadata, never the actual secret.
- **Runtime Retrieval**: When loading a macro from a remote repo, the server:
  1. Parses the macro metadata from the snapshot to get the `credentialId`
  2. Looks up the corresponding environment variable (e.g., `ETHERCALC_GIT_CRED_${credentialId}`)
  3. Uses the retrieved credential to authenticate git operations (clone, fetch, checkout specific commit)
  4. Credentials are cached in memory for the server session but never persisted to disk or logs
- **Security Measures**: 
  - Credentials are loaded at server startup and held in memory only
  - No credential exposure in API responses or error messages
  - Failed credential lookups result in macro load failure with generic error (e.g., "Repository access denied")
  - Audit logging records credential ID usage without exposing the secret
- **Integration Points**: This requires extending EtherCalc's server code (likely in `src/main.ls` or a new macro loader module) to handle git operations with authentication, using Node.js libraries like `simple-git` or `nodegit` for git interactions.

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
- Macro snapshot extension and versioned section definition are documented in this file (see Macro Storage Schema). Added `--EtherCalc-Macros--` section with version header.
- Schema and parsing for Macro storage inside snapshots documented in this file (see Macro Storage Schema) and MACRO-PLAN/03-storage-strategy.md

## Remaining todos for planning
.

