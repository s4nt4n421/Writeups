# Hack The Box — Fireflow

> **Difficulty:** Medium  
> **OS:** Linux  
> **Platform:** Hack The Box  
> **Target:** `10.129.107.65` (instance IP; do not rely on this value for a permanent writeup)

---

## 1. Overview

**Fireflow** is a Hack The Box Linux machine used for authorized lab practice.

This write-up is being built progressively from the evidence collected during the OpenCode session.  
Only verified observations are included; unresolved parts are intentionally marked as **TBD**.

### Attack Chain

```text
Reconnaissance
    ↓
Service / Web enumeration
    ↓
Initial Access
    ↓
Post-Exploitation
    ↓
Privilege Escalation
    ↓
Root
```

> **Current status:** The machine has reached the privilege-escalation stage in the OpenCode workflow. The final escalation path is still being documented.

---

# 2. Scope

### Target

```text
10.129.107.65
```

### Scope

Only the target above was used during the authorized laboratory exercise.

### Notes

- HTB lab target.
- No other hosts should be included in the attack scope.
- The assigned IP may differ between HTB instances.

---

# 3. Reconnaissance

## 3.1 Initial discovery

OpenCode started with reconnaissance of the target and progressed through the machine's exposed services.

### Evidence

The OpenCode session tracks the reconnaissance phase as:

```text
Fase 1: Reconocimiento (nmap) contra 10.129.107.65
```

### Initial service discovery

The session later interacted with the SSH service on TCP/22 during post-exploitation.

Further service/version details will be added here from the verified reconnaissance output.

**Status:** TBD — awaiting the original Nmap evidence.

---

# 4. Enumeration

The investigation proceeded from network discovery into service and web/exposed-surface enumeration.

The OpenCode workflow is configured to:

- identify services and versions using evidence;
- enumerate web/HTTPS surfaces when present;
- review authentication boundaries;
- maintain explicit hypotheses;
- validate findings before exploitation.

### Verified workflow state

The right-side task tracker in the session records:

```text
[ ] Fase 2: Enumeración de servicios web/expuestos
[ ] Revisión de fase con pentest-reviewer tras reconocimiento
[ ] Fase 3: Análisis de vulnerabilidades (security-auditor)
[ ] Fase 4: Initial Access
[ ] Fase 5: Post-explotación
[ ] Fase 6: Privilege Escalation a root
[ ] Revisión final + informe (report-writer)
```

> These are workflow states from the OpenCode session, not individual vulnerability findings. Exact technical findings will be inserted only when supported by command output.

---

# 5. Initial Access

## 5.1 Access vector

**TBD**

The final write-up will document:

1. the vulnerable surface;
2. the observed behavior;
3. the validation performed;
4. the exact command/request used;
5. the resulting access level.

### Evidence

**TBD — insert verified command output and evidence from the session.**

---

# 6. User Access

## 6.1 User flag

The OpenCode session reported the user flag as:

```text
746ff56afcc931c6dd581ce3e58db606
```

> For a public GitHub repository, flags can be removed if you prefer keeping the repository focused on methodology.

## 6.2 Access level

The session reached a user-level context before moving into privilege enumeration.

The exact account name and supporting command output should be inserted from the verified session evidence.

**Status:** Partially documented.

---

# 7. Post-Exploitation

After user access, OpenCode moved into local privilege enumeration.

The current workflow explicitly checks:

- sudo permissions;
- kernel information;
- mail;
- SUID binaries;
- other local privilege boundaries.

### Current phase

The session reports:

```text
Objetivo: identificar el vector de escalada
Impacto: solo lectura
```

It also notes the possible relevance of:

```text
/opt/lab
~/.mcp
```

The session recorded a newly created directory:

```text
~/.mcp
```

with the date shown in the OpenCode output as:

```text
2026-10-01 09:00
```

> This is an observed artifact from the session and should be retained only if it proves relevant to the final exploit chain.

---

# 8. Privilege Escalation

## 8.1 Current investigation

OpenCode has reached the privilege-escalation stage and is enumerating possible local vectors.

The current task was delegated to:

```text
@pentest-operator
```

### Current objective

```text
Identificar el vector de escalada
```

### Checks currently being considered

- `sudo -l`
- kernel information
- mail
- SUID binaries
- `/opt/lab`
- `~/.mcp`

### Example command from the session

The operator performed a local/network sanity check including:

```bash
ping -c 1 -W 3 10.129.107.65
nc -z -w 5 10.129.107.65 22
```

This was part of the operator's verification process.

> The exact privilege-escalation exploit is intentionally left **TBD** until it is confirmed from the session evidence.

---

# 9. Root

**TBD**

This section will be completed once the session verifies:

```bash
id
```

showing:

```text
uid=0(root)
```

and reads the actual `/root/root.txt` from the target.

### Required evidence

```bash
id
cat /root/root.txt
```

---

# 10. Kill Chain

The final verified chain will be documented in this form:

```text
[Recon]
   ↓
[Service Enumeration]
   ↓
[Vulnerability Discovery]
   ↓
[Validation]
   ↓
[Initial Access]
   ↓
[User]
   ↓
[Local Enumeration]
   ↓
[Privilege Escalation]
   ↓
[Root]
```

The concrete techniques will be filled only from observed evidence.

---

# 11. Commands

This section will contain the important commands actually executed during the session.

## Recon

```bash
# TBD
```

## Enumeration

```bash
# TBD
```

## Initial Access

```bash
# TBD
```

## Post-Exploitation

```bash
# TBD
```

## Privilege Escalation

```bash
# TBD
```

## Root Verification

```bash
# TBD
```

---

# 12. Findings

| Finding | Status | Evidence |
|---|---|---|
| Initial access vector | TBD | Session evidence pending |
| User access | Confirmed | User flag obtained |
| Privilege escalation vector | In progress | Local enumeration active |
| Root access | TBD | Awaiting final verification |

---

# 13. Dead Ends / Inefficient Paths

This section will document approaches that were tested and discarded.

Examples will be added only when the session provides evidence that the path was actually attempted.

---

# 14. Lessons Learned

The final lessons will focus on reusable methodology rather than the solution itself.

Expected categories:

- prioritize evidence over assumptions;
- keep an explicit hypothesis tree;
- use the smallest useful validation test;
- avoid repeating enumeration;
- estimate cost before expensive operations;
- review local privilege boundaries systematically;
- validate a CVE before attempting exploitation.

---

# 15. Defensive Recommendations

The final version will map each confirmed vulnerability to:

- root cause;
- security impact;
- mitigation;
- validation method.

**TBD** until the complete exploit chain is verified.

---

# 16. Final Summary

**Current status:** Fireflow is still being documented.

The OpenCode session has already reached the **privilege-escalation investigation** stage.  
The final write-up will be updated as the remaining evidence is collected.

### Final chain

```text
TBD → TBD → User → TBD → Root
```

---

## Disclaimer

This write-up is intended for authorized Hack The Box lab practice and educational purposes.
