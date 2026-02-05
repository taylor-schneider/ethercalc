# Sandboxing & Security Proposal

## Table of Contents
- [1) Notes](#1-notes)
- [3) Auditing & Errors](#3-auditing--errors)
- [4) Policy Hierarchy (How It Applies)](#4-policy-hierarchy-how-it-applies)
- [5) Sandboxing Configuration Settings](#5-sandboxing-configuration-settings)
  - [Runtime Trade-Offs](#runtime-trade-offs)
  - [Shared Sandboxing for UDFs and Macros](#shared-sandboxing-for-udfs-and-macros)
  - [Execution Isolation](#execution-isolation)
  - [Import Handling](#import-handling)
  - [Resource Limits](#resource-limits)
  - [Network Policy](#network-policy)
  - [Macro API Surface](#macro-api-surface)
  - [Determinism & Time](#determinism--time)
  - [Logging & Auditing](#logging--auditing)
  - [Error Handling](#error-handling)
- [6) Configuration File](#6-configuration-file)
  - [Recommended Configuration (Secure Defaults)](#recommended-configuration-secure-defaults)
  - [Unrestricted Configuration (Permissive for Development)](#unrestricted-configuration-permissive-for-development)

## 1) Notes

- Macro calls that attempt to exceed policy restrictions should fail with a clear error indicating the policy restriction 
- Macros can only read/write sheet data through the macro API.
- Any side effects are limited to the workbook (no external writes).
- Macros cannot read credentials or server-only data.
- **Import Blocking**: The current EtherCalc does not have macro execution/sandboxing implemented. The proposed sandbox (VM/isolated-vm/worker) supports blocking imports via `allowDynamicImport: false` and context restrictions. AST validation rejects code with non-whitelisted imports on commit; runtime enforcement blocks attempts at execution.

## 3) Auditing & Errors
The srver will provide an **Audit log** which details: 
- macro run attempts
- denials
- timeouts
- policy violations.
- Errors (permission denied, policy conflict, timeout, validation).

## 4) Policy Hierarchy (How It Applies)
- **Global policy**: Allow the server admin to sets the default policy for all rooms, macros, and use
- **Per-room policy**: (optional) Allows author to further restrics the policy of a specific room beyond the defaults.
- **Per-macro flags**: (optional) Allows author to further restrics the policy of a specific macro.
- **Per-user overrides** (optional): Allows admin to set new default policy for user, and effectively all the rooms/macros they are the author for.

**Policy Decision Flow Diagram**:
Below we can see how the different levels of the policy hierarchy stack on top of eachother to tighten or loosen policy restrictions.

```
Global Policy (Admin-Controlled Restrictions)
    ↓ (Most Permissive Starting Point)
Per-Room Policy (Author-Controlled Restrictions)
    ↓ (Can only tighten, not loosen)
Per-Macro Flags (Author-Controlled Restrictions)
    ↓ (Can only tighten, not loosen)
Per-User Overrides (Admin-Controlled Exceptions)
    ↓ (Can expand permissions beyond global/room level restrictions)
Final Effective Policy
```


---

## 5) Sandboxing Configuration Settings

### Runtime Trade-Offs
The default runtime is `"isolated-vm"` because it balances security, performance, and ease of use. While `"worker"` supports the most settings and offers the strongest isolation, it has higher overhead. Here's a comparison:

| Aspect | VM | Isolated-VM | Worker |
|--------|----|-------------|--------|
| **Security Level** | Least secure (may leak Node.js globals, basic isolation) | Medium secure (stronger context isolation) | Most secure (process-level isolation) |
| **Performance Overhead** | Low (fast, in-process) | Moderate (some overhead for context setup) | High (separate process/thread) |
| **Ease of Use** | Simple, but insecure | Moderate (network requires external proxying) | Complex (process management) |
| **Feature Support** | Basic (limited advanced controls) | Most features (strong import blocking, etc.) | All features (full limits) |
| **Best For** | Development/testing with low security needs | Production with good security-performance balance | High-security environments where isolation is critical |

### Shared Sandboxing for UDFs and Macros
UDFs (user-defined functions) and action macros execute in the same sandboxed environment. Both use the same `runtime` setting (VM, isolated-vm, or worker) for isolation, and all configurations in this section apply uniformly. UDFs run during formula recalc, while macros run via the `/run` endpoint, but they share security controls (e.g., resource limits, import blocking) to ensure consistency and prevent bypasses. No separate environments—UDFs aren't less sandboxed than macros.

### Execution Isolation

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| runtime | string | "isolated-vm" | Execution isolation choice: "vm" (basic), "isolated-vm" (strong), "worker" (process-level). | Yes | Yes | Yes |
| allowDynamicImport | boolean | false | Controls dynamic imports; enforced via context restrictions. | Partial (basic) | Yes | Yes |
| allowEval | boolean | false | Allows eval() execution. | Yes | Yes | Yes |
| allowProcessEnv | boolean | false | Allows access to process.env. | No | Yes | Yes |
| allowFS | boolean | false | Allows filesystem access. | No | Yes | Yes |
| allowChildProcess | boolean | false | Allows spawning child processes. | No | No | Yes |
| allowNativeModules | boolean | false | Allows loading native Node.js modules. | No | No | Yes |

### Import Handling

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| allowedModules | string[] | [] | Whitelisted module names for imports (e.g., ["lodash", "moment"]). Empty allows none. | Partial | Yes | Yes |
| importValidation | boolean | true | Enable AST validation for imports on commit. | Yes | Yes | Yes |
| importStrategy | string | "whitelist" | Import control: "whitelist" (only allowed), "blacklist" (all except blocked), "none" (no imports). | Partial | Yes | Yes |
| blockedModules | string[] | [] | Blacklisted modules if strategy is "blacklist". | Partial | Yes | Yes |

- **Import Blocking Context**: Current EtherCalc has no macro sandboxing. Proposed runtimes support blocking via settings and context control. AST validation rejects non-whitelisted imports on commit; runtime blocks execution attempts.
- **Static vs Dynamic**: `allowDynamicImport` (from Execution Isolation) controls dynamic imports; static imports validated at parse time.

### Resource Limits

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| maxRuntimeMs | number | 500 | Maximum execution time in milliseconds. | Yes | Yes | Yes |
| maxMemoryMb | number | 64 | Maximum memory usage in MB. | Partial | Yes | Yes |
| maxOutputBytes | number | 65536 | Maximum output size in bytes. | Yes | Yes | Yes |
| maxStackDepth | number | 1024 | Maximum call stack depth. | Partial | Yes | Yes |
| maxInstructionCount | number | 1000000 | Maximum instruction count (if supported). | No | Partial | Yes |

### Network Policy

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| networkEnabled | boolean | false | Enables network access. | No | Partial (requires proxy) | Yes |
| allowedHosts | string[] | [] | List of allowed hostnames or host:port. | No | Partial | Yes |
| allowedSchemes | string[] | ["https"] | Allowed URL schemes. | No | Partial | Yes |
| allowedPorts | number[] | [443] | Allowed ports. | No | Partial | Yes |
| maxNetworkRequests | number | 0 | Maximum network requests (0 when disabled). | No | Partial | Yes |
| maxNetworkBytes | number | 0 | Maximum network bytes (0 when disabled). | No | Partial | Yes |
| networkTimeoutMs | number | 1000 | Network request timeout in milliseconds. | No | Partial | Yes |

### Macro API Surface

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| allowSheetRead | boolean | true | Allows reading sheet data. | Yes | Yes | Yes |
| allowSheetWrite | boolean | true | Allows writing sheet data. | Yes | Yes | Yes |
| allowWorkbookRead | boolean | true | Allows reading workbook metadata. | Yes | Yes | Yes |
| allowWorkbookWrite | boolean | true | Allows writing workbook data. | Yes | Yes | Yes |
| allowClipboardAccess | boolean | false | Allows clipboard access. | No | Yes | Yes |
| allowMetadataRead | boolean | true | Allows reading metadata (sheet names, counts). | Yes | Yes | Yes |

### Determinism & Time

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| allowDateTime | boolean | false | Exposes Date.now() if true. | Yes | Yes | Yes |
| allowRandom | boolean | false | Exposes Math.random() if true. | Yes | Yes | Yes |
| timeZone | string | "UTC" | IANA timezone for date functions. | Yes | Yes | Yes |

### Logging & Auditing

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| auditEnabled | boolean | true | Enables audit logging. | Yes | Yes | Yes |
| auditDetailLevel | string | "standard" | Logging detail: "minimal", "standard", "verbose". | Yes | Yes | Yes |
| logMaxBytes | number | 65536 | Maximum log size in bytes. | Yes | Yes | Yes |

### Error Handling

| Configuration | Type | Default | Description | Applies to VM | Applies to Isolated-VM | Applies to Worker |
|---------------|------|---------|-------------|---------------|-------------------------|-------------------|
| onTimeout | string | "abort" | Action when macro exceeds time limit: "abort" (stop immediately, return error) or "wait" (allow additional time if under limit threshold, otherwise abort). | Yes | Yes | Yes |
| onPolicyViolation | string | "deny" | Action when macro attempts forbidden operation: "abort" (stop execution, return error) or "deny" (block the operation, allow macro to continue). | Yes | Yes | Yes |

**Note on Configurability**: Most settings can be enabled/disabled or configured, but support depends on the chosen `runtime`. For example:
- `"vm"`: Supports basic limits and isolation but may leak some Node.js globals.
- `"isolated-vm"`: Stronger isolation, supports most limits, but network access requires external proxying.
- `"worker"`: Process-level isolation, supports all limits, but higher overhead.
Not all combinations are guaranteed; test with your chosen runtime.

## 6) Configuration File
These settings map to a JSON config file, e.g., `sandbox-config.json`. Below are two examples: recommended for security and unrestricted for development.

### Recommended Configuration (Secure Defaults)
```json
{
  "runtime": "isolated-vm",
  "execution": {
    "allowDynamicImport": false,
    "allowEval": false,
    "allowProcessEnv": false,
    "allowFS": false,
    "allowChildProcess": false,
    "allowNativeModules": false
  },
  "importHandling": {
    "allowedModules": [],
    "importValidation": true,
    "importStrategy": "whitelist",
    "blockedModules": []
  },
  "resourceLimits": {
    "maxRuntimeMs": 500,
    "maxMemoryMb": 64,
    "maxOutputBytes": 65536,
    "maxStackDepth": 1024,
    "maxInstructionCount": 1000000
  },
  "network": {
    "enabled": false,
    "allowedHosts": [],
    "allowedSchemes": ["https"],
    "allowedPorts": [443],
    "maxRequests": 0,
    "maxBytes": 0,
    "timeoutMs": 1000
  },
  "api": {
    "allowSheetRead": true,
    "allowSheetWrite": true,
    "allowWorkbookRead": true,
    "allowWorkbookWrite": true,
    "allowClipboardAccess": false,
    "allowMetadataRead": true
  },
  "determinism": {
    "allowDateTime": false,
    "allowRandom": false,
    "timeZone": "UTC"
  },
  "audit": {
    "enabled": true,
    "detailLevel": "standard",
    "logMaxBytes": 65536
  },
  "errors": {
    "onTimeout": "abort",
    "onPolicyViolation": "deny"
  }
}
```

### Unrestricted Configuration (Permissive for Development)
```json
{
  "runtime": "vm",
  "execution": {
    "allowDynamicImport": true,
    "allowEval": true,
    "allowProcessEnv": true,
    "allowFS": true,
    "allowChildProcess": true,
    "allowNativeModules": true
  },
  "importHandling": {
    "allowedModules": ["*"],
    "importValidation": false,
    "importStrategy": "none",
    "blockedModules": []
  },
  "resourceLimits": {
    "maxRuntimeMs": 30000,
    "maxMemoryMb": 512,
    "maxOutputBytes": 10485760,
    "maxStackDepth": 10000,
    "maxInstructionCount": 10000000
  },
  "network": {
    "enabled": true,
    "allowedHosts": ["*"],
    "allowedSchemes": ["http", "https"],
    "allowedPorts": [80, 443],
    "maxRequests": 100,
    "maxBytes": 10485760,
    "timeoutMs": 5000
  },
  "api": {
    "allowSheetRead": true,
    "allowSheetWrite": true,
    "allowWorkbookRead": true,
    "allowWorkbookWrite": true,
    "allowClipboardAccess": true,
    "allowMetadataRead": true
  },
  "determinism": {
    "allowDateTime": true,
    "allowRandom": true,
    "timeZone": "UTC"
  },
  "audit": {
    "enabled": false,
    "detailLevel": "minimal",
    "logMaxBytes": 1048576
  },
  "errors": {
    "onTimeout": "wait",
    "onPolicyViolation": "deny"
  }
}
```
