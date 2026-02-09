# LDAP Integration for EtherCalc

## Overview

EtherCalc itself doesn't have built-in LDAP authentication, but supports it through Apache proxy configuration. Apache acts as a reverse proxy that authenticates users against LDAP before forwarding requests to EtherCalc.

## Current Apache Configuration

The existing `apache/apache-ldap.conf` provides basic access control:

- Apache authenticates users via LDAP
- Grants access based on LDAP group membership:
  - `cn=ethercalc,ou=groups,dc=example,dc=com` - access to all sheets
  - `cn=<sheetname>,cn=ethercalc,ou=groups,dc=example,dc=com` - access to specific sheets
- Acts as reverse proxy to EtherCalc running on `http://127.0.0.1:8000/`

## LDAP-to-HMAC Token Routing

The current Apache config provides basic access control but doesn't integrate with EtherCalc's HMAC token system. For full integration, additional configuration would be needed:

### 1. Apache Authentication
Apache authenticates users via LDAP and sets environment variables (e.g., `REMOTE_USER`) with the authenticated username.

### 2. Token Generation
A custom Apache module or reverse proxy logic would need to:
- Take the authenticated LDAP username
- Generate appropriate EtherCalc HMAC tokens using the same secret key as EtherCalc
- Rewrite URLs to include the correct `?auth=<hmac>` parameter

### 3. Session Management
Since EtherCalc generates random usernames per session, LDAP integration would need to decide whether to:
- Use LDAP usernames as EtherCalc usernames (replacing random ones)
- Maintain session-based random usernames but tie them to LDAP sessions
- Generate consistent usernames derived from LDAP identities

### 4. Token Lifecycle
LDAP users might generate multiple HMAC tokens for different rooms/sessions. The integration would need to handle:
- Per-room tokens (different HMAC for each spreadsheet)
- Session timeouts (LDAP sessions vs EtherCalc auth tokens)
- Token refresh when LDAP credentials are revalidated

## Implementation Status

**Current State**: Basic LDAP authentication via Apache proxy is supported but requires manual HMAC token management.

**Full Integration**: Would require custom Apache modules or reverse proxy logic to automatically generate and inject EtherCalc HMAC tokens based on LDAP authentication.

**Without Custom Integration**: LDAP users would still need to manually obtain HMAC tokens through EtherCalc's `/edit` or `/view` endpoints after LDAP authentication, or the EtherCalc server would need modification to automatically inject tokens based on LDAP authentication status.