# OS Command Injection — Medium Security

## Overview

- **Security Level:** Medium
- **Vulnerability Type:** OS Command Injection
- **Application:** Damn Vulnerable Web Application (DVWA)

## Description

OS Command Injection occurs when user-controlled input is incorporated into an operating system command in a way that allows the command's execution to be altered.

In DVWA's Medium security level, the application introduces blacklist-based filtering to remove selected shell operators. However, the filtering is incomplete, allowing command injection to remain possible.

## Source Code Analysis

The application retrieves the user-supplied input:

```php
$target = $_REQUEST['ip'];
```

It defines a blacklist that removes two shell operators:

```php
$substitutions = array(
    '&&' => '',
    ';'  => '',
);
```

The application applies the blacklist using `str_replace()`:

```php
$target = str_replace(
    array_keys($substitutions),
    $substitutions,
    $target
);
```

Finally, the filtered input is concatenated into an operating system command:

```php
$cmd = shell_exec('ping -c 4 ' . $target);
```

The Windows branch uses a different `ping` command syntax but follows the same general approach.

### Root Cause

The application relies on an incomplete blacklist and still concatenates user-controlled input directly into a shell command.

Although `&&` and `;` are removed, other shell operators are not covered by the filter. This allows specially crafted input to influence command execution.

### Key Observation

The filtering mechanism removes specific strings rather than preventing the shell from interpreting user input as part of a command.

## Payloads

The following payloads demonstrate the filtering behavior observed in the local DVWA lab.

### 1. Filtered Command Separators

```bash
127.0.0.1; whoami
127.0.0.1 && whoami
```

**Observation:** In the tested environment, these inputs did not execute `whoami` as a separate command because the blacklist removed `;` and `&&`.

After filtering, the input no longer contains the original command separator.

### 2. Conditional Execution — Filter Bypass

```bash
Invalid_IP || whoami
```

**Observation:** The `whoami` command executed in the tested environment.

**Explanation:** The `||` operator is not included in the blacklist. If the preceding command fails, the shell executes the command that follows `||`.

This demonstrates that filtering selected operators does not eliminate OS Command Injection.

### 3. System Enumeration

```bash
Invalid_IP || uname -a
Invalid_IP || hostname
Invalid_IP || id
```

These examples demonstrate how an unfiltered shell operator may be used to attempt additional system-enumeration commands when the preceding command fails.

**Note:** Conditional execution depends on the actual exit status of the preceding command. Record the observed output in the lab rather than assuming every payload will execute as intended.

## Impact

Successful exploitation may allow an attacker to:

- Execute arbitrary OS commands with the privileges of the web server process.
- Retrieve system and environment information.
- Access files permitted to the application process.
- Potentially compromise the server, depending on the execution context and permissions.

## Mitigation

- Avoid invoking OS commands when equivalent language or framework APIs are available.
- If command execution is necessary, use APIs that pass arguments separately rather than constructing shell command strings.
- Validate input using a strict allowlist appropriate to the expected data.
- Run the application with least-privilege permissions.

A blacklist that removes selected shell operators is not a reliable primary defense against OS Command Injection.
