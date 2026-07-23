# Runbook: Pushing a Local Repo to Gitea over SSH

## Prerequisites
- A repo created in the Gitea web UI
- Your SSH public key added under Gitea → Settings → SSH/GPG Keys
- Know the Gitea host's actual IP/hostname and SSH port (see note below if Gitea sits behind a reverse proxy)

## 1. Set up `.gitignore` before your first commit
Create `.gitignore` in the repo root **before** running `git add .`, so sensitive or generated files never get staged in the first place.

```bash
cd /path/to/project
nano .gitignore
```

Common entries to consider:
```gitignore
# Secrets / environment
.env
*.secret
*.key
*.pem
*.crt

# Logs
*.log

# Build/runtime artifacts
# (adjust to your stack)

# OS/editor cruft
.DS_Store
*.swp
```

## 2. Initialize the repo and commit
```bash
git init
git add .
git status   # sanity check what's staged before committing
git commit -m "Initial commit"
```

### If a sensitive file was staged before `.gitignore` existed
```bash
git rm --cached <file>
git commit -m "Stop tracking <file>"
```
If this happens **after a push**, the file is already exposed on the server — rotate any secrets it contained. Removing it from history afterward (e.g. `git filter-repo`) requires a force-push and doesn't fully undo prior exposure.

## 3. Verify SSH connectivity to Gitea
```bash
ssh -T git@<gitea-host> -p <port>
```
Expected success response:
```
Hi there, <username>! You've successfully authenticated with the key named <key-name>, but Gitea does not provide shell access.
```
This message is normal and confirms auth is working — Gitea's git user never has a shell.

**Common failure modes:**
| Symptom | Likely cause |
|---|---|
| `Connection refused` | Wrong port, or nothing listening there — check with `ss -tlnp` on the Gitea host |
| `Permission denied (publickey)` | Key mismatch — confirm the exact `.pub` content matches what's registered in Gitea; check `ssh-add -l` to see which key is being offered |
| Times out entirely | Firewall/routing between client and Gitea host is blocking the port |

### If Gitea is behind a reverse proxy (e.g. Caddy, nginx, Traefik) or on a separate network segment/VLAN
Reverse proxies built for HTTP(S) generally **do not** forward raw TCP/SSH traffic unless explicitly configured with a layer-4/stream module. If Gitea's web UI hostname resolves to the proxy rather than the Gitea host itself, SSH to that hostname will fail even though HTTP works fine.

Fix options:
- **Use the Gitea host's real IP/hostname directly** for git+SSH operations, bypassing the proxy entirely (simplest, most common)
- **Split DNS**: add a separate DNS record (e.g. `git-ssh.example.com`) that resolves straight to the Gitea host, distinct from the proxied web UI hostname
- **Layer-4 proxying**: only if you specifically want all traffic funneled through one ingress point — adds complexity and isn't necessary for a single git server

If firewall rules sit between the client and Gitea host (e.g. inter-VLAN rules on a router/firewall), confirm a rule explicitly allows the SSH port between those segments.

## 4. Add the remote and push
Use the **verified working host/port** from step 3 — not necessarily the default clone URL Gitea's web UI displays, which is generated from Gitea's configured domain and may point at a proxy.

```bash
git remote add origin ssh://git@<gitea-host>:<port>/<username>/<repo-name>.git
git remote -v          # confirm it's correct
git branch -M main     # or master, depending on convention
git push -u origin main
```

## Key takeaway
Gitea's suggested clone URL assumes SSH traffic follows the same path as the web UI. In any setup where a reverse proxy or VLAN segmentation sits in front of Gitea, that assumption breaks — SSH needs a direct path to the Gitea host itself.
