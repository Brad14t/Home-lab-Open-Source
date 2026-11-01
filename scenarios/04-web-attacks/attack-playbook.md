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

Placeholder `evidence/01-command-injection-baseline.png`.

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

Placeholder `evidence/02-command-injection-whoami.png`.

If the page returns both the `ping` output **and** the result of `whoami`,
this confirms the vulnerability: the application is building a shell command
by directly concatenating user input, with no input validation or
sanitization at all. This is a classic OS Command Injection vulnerability
(CWE-78) — anything the web server's user account can run, an attacker can
now run too.
