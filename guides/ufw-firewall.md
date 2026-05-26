# UFW Firewall — Close Everything You Did Not Open

> A Bitcoin node exposed on the internet without a firewall is a target waiting to be hit.
> SSH hardening blocks one attack vector. The firewall blocks all the others.
> This guide covers complete UFW configuration for a Bitcoin/Lightning node on Ubuntu 24.

---

## The problem with "open" nodes

When you spin up a VPS and install Bitcoin Core + LND, by default the system accepts connections on any port. Not because you chose that — it's simply the default behavior of a Linux system without an active firewall.

This means:

- Anyone can attempt connections to LND's internal ports (REST API, gRPC)
- Your auxiliary services (LNBits, Nextcloud, BTCPay) are reachable before you've even configured them
- A misconfiguration that accidentally exposes a sensitive port has no additional layer to catch it

**UFW is not an alternative to application-level hardening. It's the layer that covers mistakes you don't know you've made yet.**

---

## What runs on a typical node and what should be exposed

Before touching UFW, know what you're protecting.

| Service | Port | Exposed? | Notes |
|---|---|---|---|
| Bitcoin Core P2P | 8333 | ✅ Yes | Peers connect here |
| Bitcoin Core RPC | 8332 | ❌ No | Localhost only |
| LND P2P | 9735 | ✅ Yes | Lightning peers connect here |
| LND gRPC | 10009 | ❌ No | Localhost or private network only |
| LND REST | 8080 | ❌ No | Localhost or private network only |
| SSH | your port | ✅ Yes | See ssh-hardening.md |
| LNBits | 5000 | ❌ No | Behind reverse proxy only |
| BTCPay Server | 443 | ✅ Yes (HTTPS) | If self-hosted with a domain |
| Tor SOCKS | 9050 | ❌ No | Localhost only |
| Tor Control | 9051 | ❌ No | Localhost only |

**The rule is simple:** expose only what must be reachable from outside. Everything else goes on localhost or a private network (Tailscale, ZeroTier).

---

## Installation and initial state

On Ubuntu 24, UFW is already installed. Verify:

```bash
sudo ufw version
sudo ufw status
```

If the status is `inactive`, it's disabled. If it's `active`, check existing rules before proceeding:

```bash
sudo ufw status numbered
```

> ⚠️ **Do not enable UFW on a remote VPS until you've added the SSH rule.** You risk locking yourself out.

---

## Step 1 — Default policies

This is the correct starting point for any server:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

**Deny incoming** means all inbound traffic is blocked by default, except what you explicitly allow.

**Allow outgoing** means your node can make outbound calls freely — required for Bitcoin sync, Tor connections, and system updates.

If you want full control over outbound traffic too (very restrictive setup):

```bash
sudo ufw default deny outgoing
# Then add explicit rules for each outbound service
# Not recommended to start — makes management significantly more complex
```

---

## Step 2 — SSH first

Before enabling UFW, always add your SSH rule first. If you followed `ssh-hardening.md` and use a non-standard port:

```bash
# Replace 2229 with your actual SSH port
sudo ufw allow 2229/tcp comment 'SSH'
```

If you're still on port 22 (change it — see ssh-hardening.md):

```bash
sudo ufw allow 22/tcp comment 'SSH default'
```

> If you lose SSH access on a VPS, most providers have an emergency console. But it's an avoidable hassle.

---

## Step 3 — Enable UFW

With the SSH rule in place:

```bash
sudo ufw enable
```

The system asks for confirmation. Type `y`. UFW is now active and persists across reboots.

Immediate verification:

```bash
sudo ufw status verbose
```

You should see:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
2229/tcp                   ALLOW IN    Anywhere
```

---

## Step 4 — Bitcoin Core ports

```bash
# P2P mainnet — required to connect to peers
sudo ufw allow 8333/tcp comment 'Bitcoin Core P2P mainnet'

# If you also run testnet
# sudo ufw allow 18333/tcp comment 'Bitcoin Core P2P testnet'

# If you run signet
# sudo ufw allow 38333/tcp comment 'Bitcoin Core P2P signet'
```

**Do not open the RPC port (8332).** Bitcoin Core RPC has authentication, but exposing it to the internet is unnecessary risk. Access it only from localhost or via SSH tunnel.

Verify RPC is not already exposed:

```bash
ss -tlnp | grep 8332
# Must show 127.0.0.1:8332, not 0.0.0.0:8332
```

If you see `0.0.0.0:8332`, check your `bitcoin.conf` — add:

```
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
```

---

## Step 5 — LND ports

```bash
# P2P Lightning — required for peers
sudo ufw allow 9735/tcp comment 'LND P2P'
```

**Do not open LND's internal ports:**

```bash
# These must NOT be opened to the internet:
# 10009/tcp — gRPC (used by lncli, mobile apps, etc.)
# 8080/tcp  — REST API
```

LND's gRPC (10009) and REST API (8080) carry the `admin.macaroon`. Anyone who reaches them controls your node. Use them only via localhost or a private VPN.

If you need remote gRPC access (e.g. from a mobile app), use an SSH tunnel:

```bash
# From your local machine, open a tunnel to the node
ssh -L 10009:localhost:10009 -p 2229 user@your-node-ip -N
# The app then connects to localhost:10009
```

---

## Step 6 — HTTPS for web services (optional)

If you run BTCPay Server, a reverse proxy (nginx/Caddy), or any web service with a domain:

```bash
sudo ufw allow 80/tcp comment 'HTTP (redirect to HTTPS)'
sudo ufw allow 443/tcp comment 'HTTPS'
```

**Do not expose services directly on their native port** (LNBits on 5000, ThunderHub on 3000, etc.). Always put them behind a reverse proxy with HTTPS. The only exposed port should be 443.

---

## Step 7 — Brute force protection with rate limiting

UFW has native rate limiting. Useful for SSH even if you already run Fail2ban:

```bash
# Replace 2229 with your SSH port
sudo ufw limit 2229/tcp comment 'SSH rate limit'
```

This automatically blocks IPs that make more than 6 connections in 30 seconds. It's an additional protection layer, not a replacement for Fail2ban.

---

## Step 8 — IPv6

UFW by default creates parallel rules for both IPv4 and IPv6. Verify this is the case:

```bash
grep IPV6 /etc/default/ufw
# Must show IPV6=yes
```

If you don't use IPv6 and want to disable it entirely at the UFW level:

```bash
sudo nano /etc/default/ufw
# Change IPV6=yes to IPV6=no
sudo ufw reload
```

> Note: disabling IPv6 in UFW does not disable it at the kernel level. To do that completely, add to `/etc/sysctl.d/99-disable-ipv6.conf`:
> ```
> net.ipv6.conf.all.disable_ipv6 = 1
> net.ipv6.conf.default.disable_ipv6 = 1
> ```
> Then: `sudo sysctl -p /etc/sysctl.d/99-disable-ipv6.conf`

---

## Step 9 — Whitelist trusted IPs (optional but recommended)

If you always access your node from the same IP (home, fixed VPN), restrict SSH to that IP:

```bash
# First remove the generic SSH rule
sudo ufw delete allow 2229/tcp

# Then add only for your IP
sudo ufw allow from 1.2.3.4 to any port 2229 proto tcp comment 'SSH from home'
```

For gRPC access from a trusted IP (e.g. your home server):

```bash
sudo ufw allow from 1.2.3.4 to any port 10009 proto tcp comment 'LND gRPC from home server'
```

---

## Step 10 — Complete setup for a Bitcoin/Lightning node

Full command sequence for a standard node. Adapt ports to your services:

```bash
# Full reset (be careful on active servers)
# sudo ufw reset

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH (use your port)
sudo ufw allow 2229/tcp comment 'SSH'

# Bitcoin Core
sudo ufw allow 8333/tcp comment 'Bitcoin Core P2P'

# LND
sudo ufw allow 9735/tcp comment 'LND P2P'

# If you have BTCPay or other HTTPS services
# sudo ufw allow 80/tcp comment 'HTTP'
# sudo ufw allow 443/tcp comment 'HTTPS'

# Rate limiting on SSH
sudo ufw limit 2229/tcp

# Enable
sudo ufw enable

# Verify
sudo ufw status numbered
```

---

## Managing rules

### View rules with numbering

```bash
sudo ufw status numbered
```

Example output:

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 2229/tcp                   LIMIT IN    Anywhere
[ 2] 8333/tcp                   ALLOW IN    Anywhere
[ 3] 9735/tcp                   ALLOW IN    Anywhere
[ 4] 2229/tcp (v6)              LIMIT IN    Anywhere (v6)
[ 5] 8333/tcp (v6)              ALLOW IN    Anywhere (v6)
[ 6] 9735/tcp (v6)              ALLOW IN    Anywhere (v6)
```

### Delete a rule by number

```bash
sudo ufw delete 3
# Deletes rule number 3 (LND P2P in this example)
```

### Delete a rule by definition

```bash
sudo ufw delete allow 9735/tcp
```

### Reload after changes

```bash
sudo ufw reload
```

### Temporarily disable (without losing rules)

```bash
sudo ufw disable
# When done:
sudo ufw enable
```

---

## Audit: what is actually exposed

After configuring UFW, verify what is genuinely reachable from outside.

### Check listening ports on the server

```bash
ss -tlnp
```

The output shows every listening service. `Local Address` column: if you see `0.0.0.0:PORT` or `*:PORT`, that port is reachable from the internet (unless UFW blocks it). If you see `127.0.0.1:PORT`, it's localhost only — correct.

### Verify what passes through UFW

```bash
sudo ufw status verbose
```

### External scan to confirm

From your local machine, scan the node to verify what is visible from outside:

```bash
# Install nmap locally if you don't have it
nmap -p 1-10000 your-node-ip
```

The only ports that should show as `open` are the ones you explicitly allowed. Everything else must be `filtered` (blocked by UFW) or `closed` (no service listening).

### Check UFW logs

```bash
sudo journalctl -k | grep UFW
# Or
sudo tail -f /var/log/ufw.log
```

Logs show every blocked connection attempt. Useful to see who is probing your node and on which ports.

---

## Logging levels

UFW has three log levels:

```bash
sudo ufw logging off      # No logging
sudo ufw logging low      # Default — logs blocked connections only
sudo ufw logging medium   # Logs blocks + some allowed connections
sudo ufw logging high     # Everything — very verbose
```

For a production node, `low` is sufficient. Use `medium` during initial setup to verify you're not accidentally blocking something required.

---

## Common mistakes

**"I enabled UFW and the node stopped syncing"**

You blocked port 8333 or Bitcoin Core's outbound connections. Check:

```bash
sudo ufw status | grep 8333
# Must show ALLOW

# Verify outbound is permitted
sudo ufw status verbose | grep outgoing
# Must show allow (outgoing)
```

**"LNBits stopped responding after enabling UFW"**

LNBits runs on an internal port (5000 by default). If you were accessing it directly via `http://your-ip:5000`, it's now blocked. Correct solution: put a reverse proxy (nginx or Caddy) in front and access it via HTTPS on port 443.

**"Fail2ban is no longer banning IPs"**

Fail2ban interacts with iptables/nftables, not UFW directly. On Ubuntu 24, verify Fail2ban is configured to use the correct backend:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add if not present:

```
[DEFAULT]
banaction = ufw
```

Then restart:

```bash
sudo systemctl restart fail2ban
```

---

## Complete UFW state — reference

This is the final rule state for a standard Bitcoin/Lightning node on Ubuntu 24:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
2229/tcp                   LIMIT IN    Anywhere          # SSH
8333/tcp                   ALLOW IN    Anywhere          # Bitcoin P2P
9735/tcp                   ALLOW IN    Anywhere          # LND P2P
2229/tcp (v6)              LIMIT IN    Anywhere (v6)
8333/tcp (v6)              ALLOW IN    Anywhere (v6)
9735/tcp (v6)              ALLOW IN    Anywhere (v6)
```

Everything else — Bitcoin RPC (8332), LND gRPC (10009), LND REST (8080), Tor (9050/9051), databases, internal services — does not appear here. It is not exposed.

---

## What this protects against

| Attack vector | Protected |
|---|---|
| Automated port scans | ✅ Only explicit ports visible |
| Direct access to LND gRPC/REST | ✅ Blocked by UFW |
| Direct access to Bitcoin RPC | ✅ Localhost only |
| SSH brute force | ✅ Rate limiting + Fail2ban |
| Internal services accidentally exposed | ✅ Deny incoming by default |
| Unauthorized inbound connections | ✅ Default deny policy |

## What this does NOT protect against

- **Vulnerabilities in services you exposed** — if LND 9735 has a bug, UFW won't save you
- **Localhost traffic** — processes on the server still communicate freely via 127.0.0.1
- **Application-layer attacks** — SQLi, XSS, etc. on exposed web services
- **Physical access to the server** — irrelevant if someone has the machine in their hands

---

## Bottom line

A node without a firewall is a node that trusts everything. `deny incoming` as a default means every port is closed until you open it — not the other way around.

Open 8333 for Bitcoin. Open 9735 for Lightning. Keep gRPC, REST, RPC, and everything else closed. Then verify with nmap that what you see matches what you decided.

The firewall does not protect you from everything. It protects you from the mistakes you don't know you've made yet.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
