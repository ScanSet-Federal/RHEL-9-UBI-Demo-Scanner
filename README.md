# ICS Learning Guide - RHEL Scanner Implementation

## Overview
This scanner implements a subset of the **Infrastructure Compliance Syntax (ICS)** language, providing automated compliance validation for RHEL 9 systems.  
It demonstrates the core ICS concepts through working CTN (Criterion Type Node) implementations.

## Requirements
VSCode with Devcontainers

---

## What is ICS?
Infrastructure Compliance Syntax is a domain-specific language for expressing compliance policies as executable code.  
Instead of writing prose documents describing security requirements, ICS allows you to define machine-readable, automatically-validated compliance criteria.

**Key Concept:** ICS separates **"what to check"** (policy) from **"how to check it"** (implementation).  
The same ICS policy can run on different platforms with different scanners.

---

## How to run scanner
### Single File Mode

```bash
./ics-scanner file.ics
```

### Directory Batching Mode

```bash
./ics-scanner ics/
```

## ICS Components Implemented

### ✅ Fully Implemented
| Component | Purpose | Status |
|-----------|---------|--------|
| META      | Policy metadata and context | ✅ Complete |
| VAR       | Variable definitions | ✅ Complete |
| OBJECT    | Target resources to validate | ✅ Complete |
| STATE     | Expected configuration state | ✅ Complete |
| CTN       | Criterion type (validation strategy) | ✅ 6 types implemented |
| CRI       | Criteria container (AND/OR/NOT logic) | ✅ Complete |
| TEST      | Existence and item checks | ✅ All operators |


## Available CTN Types

### 1. `file_metadata` – File System Metadata Validation
**Purpose:** Fast validation of file permissions, ownership, and attributes without reading file content.

**OBJECT Example:**
```ics
OBJECT example_file
    path VAR file_path          # Required - file system path
    type `file`                 # Optional - informational
OBJECT_END
```

**STATE Fields:**

| Field       | Type    | Operations                   | Description |
|-------------|---------|-----------------------------|-------------|
| permissions | string  | =, !=                       | Octal format (e.g., 0440) |
| owner       | string  | =, !=                       | UID as string (e.g., 0 for root) |
| group       | string  | =, !=                       | GID as string |
| exists      | boolean | =, !=                       | Whether file exists |
| readable    | boolean | =, !=                       | Whether process can read |
| size        | int     | =, !=, >, <, >=, <=         | File size in bytes |

**Example:**
```ics
VAR secure_file string `/etc/shadow`
VAR root_user string `0`

OBJECT shadow_file
    path VAR secure_file
OBJECT_END

STATE secure_permissions
    permissions string = `0000`
    owner string = VAR root_user
    exists boolean = true
STATE_END

CTN file_metadata
    TEST all all
    STATE_REF secure_permissions
    OBJECT_REF shadow_file
CTN_END
```

**Field Mappings:**
- permissions → file_mode
- owner → file_owner
- group → file_group
- size → file_size

---

### 2. `file_content` – File Content Validation
**Purpose:** String-based validation of file contents.

**OBJECT Example:**
```ics
OBJECT config_file
    path VAR file_path          # Required
    type `file`                 # Optional
OBJECT_END
```

**STATE Fields:**
| Field   | Type   | Operations | Description |
|---------|--------|------------|-------------|
| content | string | =, !=, contains, not_contains, starts, ends, pattern_match | File content as UTF-8 |

**Example:**
```ics
VAR sudoers_path string `/etc/sudoers`

OBJECT sudoers_file
    path VAR sudoers_path
OBJECT_END

STATE no_nopasswd
    content string not_contains `NOPASSWD`
STATE_END

STATE has_logging
    content string contains `logfile=`
STATE_END

CTN file_content
    TEST all all
    STATE_REF no_nopasswd
    STATE_REF has_logging
    OBJECT_REF sudoers_file
CTN_END
```

**Field Mappings:**
- content → file_content

---

### 3. `rpm_package` – RPM Package Validation
**Purpose:** Verify package installation status and versions.

**OBJECT Example:**
```ics
OBJECT security_package
    package_name VAR pkg_name   # Required - package name
    type `rpm`                  # Optional
OBJECT_END
```

**STATE Fields:**
| Field     | Type    | Operations           | Description |
|-----------|---------|----------------------|-------------|
| installed | boolean | =, !=                | Whether package is installed |
| version   | string  | =, !=, >, <, >=, <=  | Package version (lexicographic) |

**Example:**
```ics
VAR openssl_pkg string `openssl`
VAR min_version string `3.0.7`

OBJECT openssl_package
    package_name VAR openssl_pkg
OBJECT_END

STATE installed_and_current
    installed boolean = true
    version string >= VAR min_version
STATE_END

CTN rpm_package
    TEST all all
    STATE_REF installed_and_current
    OBJECT_REF openssl_package
CTN_END
```

**Field Mappings:**
- installed → installed
- version → version

**Collection:** Uses `rpm -q <package>` or `rpm -qa`.

---

### 4. `systemd_service` – Systemd Service Status
**Purpose:** Validate service state (active, enabled, loaded).

**OBJECT Example:**
```ics
OBJECT critical_service
    service_name VAR svc_name   # Required - include .service suffix
    type `systemd`              # Optional
OBJECT_END
```

**STATE Fields:**
| Field   | Type    | Operations | Description |
|---------|---------|------------|-------------|
| active  | boolean | =, !=      | Service is running |
| enabled | boolean | =, !=      | Service starts at boot |
| loaded  | boolean | =, !=      | Service unit is loaded |

**Example:**
```ics
VAR sshd_service string `sshd.service`

OBJECT sshd
    service_name VAR sshd_service
OBJECT_END

STATE must_be_running
    active boolean = true
    enabled boolean = true
STATE_END

CTN systemd_service
    TEST all all
    STATE_REF must_be_running
    OBJECT_REF sshd
CTN_END
```

**Field Mappings:**
- active → active
- enabled → enabled
- loaded → loaded

**Collection:** Uses `systemctl is-active` and `systemctl is-enabled`.

---

### 5. `sysctl_parameter` – Kernel Parameters
**Purpose:** Validate kernel runtime parameters.

**OBJECT Example:**
```ics
OBJECT kernel_param
    parameter_name VAR param    # Required - dot-notation
    type `sysctl`               # Optional
OBJECT_END
```

**STATE Fields:**
| Field    | Type | Operations           | Description |
|----------|------|----------------------|-------------|
| value    | string | =, !=              | Parameter value as string |
| value_int| int    | =, !=, >, <, >=, <=| Parameter value as integer |

**Example:**
```ics
VAR ip_forward_param string `net.ipv4.ip_forward`
VAR disabled_value string `0`

OBJECT ip_forward
    parameter_name VAR ip_forward_param
OBJECT_END

STATE forwarding_disabled
    value string = VAR disabled_value
STATE_END

CTN sysctl_parameter
    TEST all all
    STATE_REF forwarding_disabled
    OBJECT_REF ip_forward
CTN_END
```

**Field Mappings:**
- value → value
- value_int → value_int

**Collection:** Uses `sysctl -n <parameter>`.

---

### 6. `selinux_status` – SELinux Enforcement Mode
**Purpose:** Verify SELinux is configured correctly.

**OBJECT Example:**
```ics
OBJECT selinux_system
    check_type `enforcement`
    type `selinux`
OBJECT_END
```

**STATE Fields:**
| Field     | Type    | Operations | Description |
|-----------|---------|------------|-------------|
| mode      | string  | =, !=      | Enforcing, Permissive, or Disabled |
| enforcing | boolean | =, !=      | Whether mode is Enforcing |

**Example:**
```ics
VAR required_mode string `Enforcing`

OBJECT selinux_check
    check_type `enforcement`
OBJECT_END

STATE must_be_enforcing
    mode string = VAR required_mode
    enforcing boolean = true
STATE_END

CTN selinux_status
    TEST all all
    STATE_REF must_be_enforcing
    OBJECT_REF selinux_check
CTN_END
```

**Field Mappings:**
- mode → mode
- enforcing → enforcing

**Collection:** Uses `getenforce`.

---

## TEST Specifications

**Syntax:**  
```
TEST <existence_check> <item_check>
```

**Existence Checks:**
| Operator   | Meaning |
|------------|---------|
| all        | All specified objects must exist |
| any        | At least one object must exist |
| at_least_one | Synonym for any |
| only_one   | Exactly one object must exist |
| none       | No objects should exist |

**Item Checks:**
| Operator   | Meaning |
|------------|---------|
| all        | All objects must pass state validation |
| any        | At least one object must pass |
| at_least_one | Synonym for any |
| only_one   | Exactly one object must pass |
| none       | No objects should pass |

**Common Patterns:**
```ics
# All packages must exist and be installed
TEST all all

# At least one firewall service is running
TEST any all

# Exactly one sshd config file exists
TEST only_one all

# Dangerous services must not be running
TEST all none

# At least one backup exists, any state passes
TEST at_least_one any
```

---

## CRI Logic (AND/OR/NOT)
Default: Multiple CTNs in a CRI are implicitly **AND**’d.

**Example:**
```ics
CRI
    CTN rpm_package ... CTN_END
    CTN file_metadata ... CTN_END
CRI_END
```

**Explicit operators:**
```ics
CRI_OR
    CTN rpm_package ... CTN_END
    CTN file_metadata ... CTN_END
CRI_END

CRI_NOT
    CTN systemd_service ... CTN_END
CRI_END
```

**Nesting Example:**
```ics
CRI_AND
    CTN rpm_package ... CTN_END

    CRI_OR
        CTN systemd_service ... CTN_END
        CTN file_metadata ... CTN_END
    CRI_END
CRI_END
```

---

## Running the Scanner
```bash
# Basic scan
./file-scanner policy.ics

# Output to file
./file-scanner policy.ics > results.json

# Verbose mode (debug)
RUST_LOG=debug ./file-scanner policy.ics
```

---

## Understanding Results

**Compliant scan:**
```json
{
  "results": {
    "check": {
      "status": "compliant",
      "pass_percentage": 100.0
    },
    "findings": [],
    "passed": true
  }
}
```

**Non-compliant scan:**
```json
{
  "results": {
    "check": {
      "status": "noncompliant",
      "pass_percentage": 66.67
    },
    "findings": [
      {
        "severity": "high",
        "title": "file_metadata validation failed",
        "description": "Field 'permissions' failed: expected '0440', got '0644'",
        "expected": { "permissions": "0440" },
        "actual": { "permissions": "0644" }
      }
    ],
    "passed": false
  }
}
```

---

## Learning Exercises
1. **Basic File Check** – Validate `/etc/passwd` exists and is readable.  
2. **Package Validation** – Check that `bash` and `coreutils` are installed.  
3. **Service Management** – Verify `sshd.service` is active and enabled, but `bluetooth.service` is disabled.  
4. **Multi-CTN Policy** – Combine file, package, and service checks in a single policy.  
5. **Complex Logic** – Use CRI_OR to check that either `firewalld` OR `iptables` is running.

---

## Common Pitfalls
- Forgetting `.service` suffix (`sshd.service` not just `sshd`)
- Wrong permission format (`0440` not `440`)
- String vs int comparisons (sysctl value as string vs int)
- Missing VAR resolution (`VAR variable_name` required)
- Container limitations (`systemctl` and `sysctl` may not work in containers)

---

## Architecture Notes
**Collection Strategy:**  
- `file_metadata` and `file_content` → FileSystemCollector  
- `rpm_package`, `systemd_service`, `sysctl_parameter`, `selinux_status` → CommandCollector  

**Execution Flow:**  
1. Parse ICS file → AST  
2. Resolve variables  
3. Build execution DAG  
4. Collect data for each OBJECT  
5. Validate against STATE requirements  
6. Apply TEST logic  
7. Aggregate results with CRI logic  
8. Generate findings  

**Performance:**  
- File metadata: ~5ms per file  
- File content: ~50ms per file (size-dependent)  
- RPM queries: ~100ms (batched when possible)  
- Service checks: ~50ms per service  
- Sysctl: ~30ms per parameter  

---

**Coverage:** ~60% of RHEL 9 STIG requirements.  
**Missing coverage:** Audit rules (`auditctl`), user/group validation (`getent`), and complex record parsing.
