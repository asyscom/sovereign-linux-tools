# Fail2ban — Intrusion Prevention for Bitcoin and Lightning Nodes

> UFW closes the ports you did not open. Fail2ban watches the ports you did.
> This guide covers complete Fail2ban configuration for a Bitcoin/Lightning node on Ubuntu 24,
> including custom jails for SSH, LND REST, and Bitcoin RPC abuse detection.

---

## The problem Fail2ban solves

A firewall with `deny incoming` blocks unauthorized ports. But the ports you deliberately opened — SSH, 8333, 9735 — are visible to the entire internet. Every day, automated scanners probe them looking for weak credentials, vulnerable daemons, or misconfigured services.

Without Fail2ban:
- SSH on any port gets continuous brute-force attempts
- A single attacker can try thousands of passwords before you notice
- Failed attempts consume resources and fill logs with noise
- There is no automatic response — detection and reaction are manual

**Fail2ban reads your service logs, detects repeated failures, and instructs UFW to ban the offending IP automatically.** The ban is temporary by default, permanent if you configure recidive jails.

---

## How it works

Three components work together:

**Filters** — regex patterns that match malicious events in log files. Located in `/etc/fail2ban/filter.d/`. Each filter defines what "bad behavior" looks like for a specific service.

**Jails** — bind a filter to a log file and a firewall action. A jail says: "watch this log, use this filter, ban IPs that match more than N times in T minutes."

**Actions** — how the ban is applied. On Ubuntu 24 with UFW, the action is `ufw` — Fail2ban calls `ufw insert 1 deny from <IP>` to block the attacker at the firewall level.

---

## Installation

```bash
sudo apt update
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

Verify it is running:

```bash
sudo fail2ban-client status
```

Output should show:

```
Status
|- Number of jail:      0
`- Jail list:
```

Zero jails at this point — that is expected. Nothing is configured yet.

---

## Step 1 — Never edit jail.conf

Fail2ban ships with `/etc/fail2ban/jail.conf`. This file is overwritten on every package update. **Never edit it.**

All configuration goes in `/etc/fail2ban/jail.local`, which overrides `jail.conf` without touching it:

```bash
sudo nano /etc/fail2ban/jail.local
```

Start with an empty file — you will build it step by step.

---

## Step 2 — Global defaults

This section applies to all jails unless overridden per-jail:

```ini
[DEFAULT]

# Your IPs — never banned under any circumstance
# Add your home IP, VPN IP, Tailscale range if you use it
ignoreip = 127.0.0.1/8 ::1

# How long an IP stays banned
# 28d = 28 days. Aggressive but effective for a node.
bantime = 28d

# Time window to count failures
findtime = 10m

# Number of failures before ban
maxretry = 5

# Incremental ban: each re-offense multiplies bantime
bantime.increment = true
bantime.factor = 2

# Use systemd journal — correct for Ubuntu 24
backend = systemd

# UFW integration — required if you followed ufw-firewall.md
banaction = ufw
banaction_allports = ufw

# Log target
logtarget = /var/log/fail2ban.log
```

**bantime = 28d** is intentionally aggressive. A legitimate user who mistyped their password can wait 28 days or contact you. An attacker should not get another chance for a month. Adjust down to `1h` or `24h` if you find it too strict.

**bantime.increment** means: first offense = 28d, second offense = 56d, third = 112d. Recidivists get progressively longer bans.

---

## Step 3 — SSH jail

The most important jail. Add this after `[DEFAULT]`:

```ini
[sshd]
enabled = true

# Match your actual SSH port from ssh-hardening.md
port = 2229

# Filter ships with Fail2ban — no custom filter needed
filter = sshd

# Ubuntu 24 logs SSH via journald
backend = systemd

# Tighter than default — SSH brute force is fast
maxretry = 3
findtime = 5m
bantime = 28d
```

> If you have not changed your SSH port yet, read `ssh-hardening.md` first. Running SSH on port 22 with Fail2ban is better than nothing, but changing the port eliminates ~99% of automated scanners before Fail2ban even needs to act.

---

## Step 4 — Recidive jail

Catches IPs that keep coming back after bans expire. Uses Fail2ban's own log as input:

```ini
[recidive]
enabled = true

filter = recidive
logpath = /var/log/fail2ban.log

# 1 year ban for repeat offenders
bantime = 365d
findtime = 1d

# Ban after being banned 3 times within 1 day
maxretry = 3
```

This jail is the long-term memory of your ban list. An IP that gets banned three separate times in a day gets a year-long ban applied automatically.

---

## Step 5 — Custom filter for LND REST API

LND's REST API (port 8080) should never be exposed to the internet — but if you accidentally left it open or are running it behind a reverse proxy, brute-force attempts on the API are a real threat.

LND logs failed authentication attempts. Create a custom filter:

```bash
sudo nano /etc/fail2ban/filter.d/lnd-rest.conf
```

```ini
[Definition]

# Match LND REST authentication failures
# LND logs these as "gRPC request failed" or "invalid macaroon"
failregex = .*invalid macaroon.*<HOST>
            .*authentication failure.*<HOST>
            .*permission denied.*<HOST>

ignoreregex =
```

Then add the jail in `jail.local`:

```ini
[lnd-rest]
enabled = true

filter = lnd-rest
port = 8080

# LND log path — adjust to your setup
logpath = /home/bitcoin/.lnd/logs/bitcoin/mainnet/lnd.log

maxretry = 3
findtime = 5m
bantime = 28d
```

> Note: if LND REST is properly locked behind UFW (port 8080 closed), this jail is a defense-in-depth layer for the case where something bypasses the firewall — reverse proxy misconfiguration, Tailscale exposure, etc.

---

## Step 6 — Custom filter for Bitcoin Core RPC

Bitcoin Core RPC (port 8332) should only be on localhost. But if it is accidentally exposed, failed RPC authentication leaves traces in debug.log:

```bash
sudo nano /etc/fail2ban/filter.d/bitcoind-rpc.conf
```

```ini
[Definition]

# Bitcoin Core logs RPC auth failures
failregex = ThreadRPCServer incorrect password attempt from <HOST>

ignoreregex =
```

Jail configuration:

```ini
[bitcoind-rpc]
enabled = true

filter = bitcoind-rpc
port = 8332

# Bitcoin Core debug log
logpath = /home/bitcoin/.bitcoin/debug.log

maxretry = 3
findtime = 10m
bantime = 28d
```

---

## Step 7 — Complete jail.local

This is the full file for a standard Bitcoin/Lightning node on Ubuntu 24:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1
bantime = 28d
findtime = 10m
maxretry = 5
bantime.increment = true
bantime.factor = 2
backend = systemd
banaction = ufw
banaction_allports = ufw
logtarget = /var/log/fail2ban.log

[sshd]
enabled = true
port = 2229
filter = sshd
backend = systemd
maxretry = 3
findtime = 5m
bantime = 28d

[recidive]
enabled = true
filter = recidive
logpath = /var/log/fail2ban.log
bantime = 365d
findtime = 1d
maxretry = 3

[lnd-rest]
enabled = true
filter = lnd-rest
port = 8080
logpath = /home/bitcoin/.lnd/logs/bitcoin/mainnet/lnd.log
maxretry = 3
findtime = 5m
bantime = 28d

[bitcoind-rpc]
enabled = true
filter = bitcoind-rpc
port = 8332
logpath = /home/bitcoin/.bitcoin/debug.log
maxretry = 3
findtime = 10m
bantime = 28d
```

Apply the configuration:

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status
```

You should now see all enabled jails listed:

```
Status
|- Number of jail:      4
`- Jail list:   bitcoind-rpc, lnd-rest, recidive, sshd
```

---

## Managing bans

### Check active bans per jail

```bash
# SSH jail
sudo fail2ban-client status sshd

# All jails at once
sudo fail2ban-client status
```

Output for a jail shows:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed: 47
|  `- File list: /var/log/auth.log
`- Actions
   |- Currently banned: 3
   |- Total banned: 12
   `- Banned IP list: 1.2.3.4 5.6.7.8 9.10.11.12
```

### Unban an IP manually

If you accidentally lock yourself out:

```bash
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

Or unban from all jails at once:

```bash
sudo fail2ban-client unban 1.2.3.4
```

### Ban an IP manually

```bash
sudo fail2ban-client set sshd banip 1.2.3.4
```

### View all currently banned IPs across all jails

```bash
sudo fail2ban-client banned
```

### Flush all bans (use with caution)

```bash
sudo fail2ban-client unban --all
```

---

## Monitoring

### Watch the log in real time

```bash
sudo tail -f /var/log/fail2ban.log
```

Every ban and unban is logged here with timestamp, jail name, and IP:

```
2026-06-03 14:22:11,847 fail2ban.actions [1245]: NOTICE  [sshd] Ban 185.220.101.47
2026-06-03 14:22:11,901 fail2ban.actions [1245]: NOTICE  [sshd] Ban 45.33.32.156
```

### Check how many IPs are currently banned

```bash
sudo fail2ban-client banned | python3 -c "
import sys, json
data = json.load(sys.stdin)
total = sum(len(v) for d in data for v in d.values())
print(f'Total banned IPs: {total}')
"
```

### Check if a specific IP is banned

```bash
sudo fail2ban-client get sshd banip | grep 1.2.3.4
```

### Weekly ban report (add to cron)

```bash
sudo crontab -e
# Add:
0 9 * * 1 fail2ban-client status sshd | grep "Total banned" >> /var/log/fail2ban-weekly.log
```

---

## Whitelist your own IP

If you always connect from the same IP (home, fixed VPN), add it to `ignoreip` in `[DEFAULT]`:

```ini
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 YOUR.HOME.IP.HERE
```

Then reload:

```bash
sudo fail2ban-client reload
```

**This is the most important setting to get right.** If you fat-finger your SSH password three times from home without a whitelist, you ban yourself for 28 days.

---

## Integration with UFW

If you followed `ufw-firewall.md`, Fail2ban needs to be told to use UFW as the ban backend. This is already in the `[DEFAULT]` section above:

```ini
banaction = ufw
banaction_allports = ufw
```

Verify that Fail2ban is correctly calling UFW when it bans an IP:

```bash
# Trigger a test ban (replace with a test IP you control)
sudo fail2ban-client set sshd banip 192.0.2.1

# Check UFW rules — the ban should appear at the top
sudo ufw status numbered | head -20

# Should show something like:
# [ 1] Anywhere                   DENY IN     192.0.2.1

# Clean up test ban
sudo fail2ban-client set sshd unbanip 192.0.2.1
```

---

## Testing the configuration

Before relying on Fail2ban in production, test that it actually works.

### Test filter regex against a log file

```bash
sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
```

Output shows how many lines matched. If 0 matches, the filter is not reading the log correctly — check the `logpath` and `backend` settings.

### Test a custom filter

```bash
sudo fail2ban-regex /home/bitcoin/.bitcoin/debug.log /etc/fail2ban/filter.d/bitcoind-rpc.conf
```

### Simulate SSH failures and verify ban

From a second machine (not your whitelisted IP):

```bash
# Attempt SSH login with wrong password 4 times
ssh -p 2229 wronguser@your-node-ip
# Repeat 4 times

# On the node, check if the IP was banned
sudo fail2ban-client status sshd
```

---

## Troubleshooting

**"Fail2ban is running but not banning anyone"**

Check two things: backend and logpath.

```bash
# Verify backend — on Ubuntu 24, systemd is correct
sudo fail2ban-client get sshd backend

# Verify the log is being read
sudo fail2ban-client get sshd logpath
```

If `backend = systemd` is set but logs are file-based:

```bash
# Check where SSH actually logs
sudo journalctl -u ssh --no-pager | tail -20
# If you see output here, systemd backend is correct
```

**"UFW rules are not being created when Fail2ban bans"**

The `banaction = ufw` setting requires UFW to be active. Verify:

```bash
sudo ufw status
# Must show: Status: active

# Also verify the Fail2ban UFW action file exists
ls /etc/fail2ban/action.d/ufw.conf
```

If `ufw.conf` is missing:

```bash
sudo apt install --reinstall fail2ban
```

**"I banned myself"**

If you are locked out via SSH, use your VPS provider's emergency console to access the server, then:

```bash
sudo fail2ban-client unban YOUR.IP.ADDRESS
# Or unban from all jails:
sudo fail2ban-client unban --all
```

Then add your IP to `ignoreip` before reloading.

**"The lnd-rest or bitcoind-rpc jail shows 0 matches"**

The log paths may differ depending on your installation method (manual, Umbrel, RaspiBlitz, etc.). Find the correct paths:

```bash
# LND log
find /home -name "lnd.log" 2>/dev/null
find /var -name "lnd.log" 2>/dev/null

# Bitcoin Core log
find /home -name "debug.log" 2>/dev/null
find /var -name "debug.log" 2>/dev/null
```

Update `logpath` in `jail.local` accordingly, then restart:

```bash
sudo systemctl restart fail2ban
```

---

## What this protects against

| Attack vector | Protected |
|---|---|
| SSH brute force | ✅ Banned after 3 failures in 5 minutes |
| Credential stuffing on SSH | ✅ Same — rapid failures trigger ban |
| LND REST API abuse | ✅ Custom filter catches auth failures |
| Bitcoin RPC unauthorized access | ✅ Custom filter catches password attempts |
| Repeat offenders after ban expiry | ✅ Recidive jail escalates to 1-year bans |
| Your own IP accidentally banned | ✅ ignoreip whitelist |

## What this does NOT protect against

- **Slow brute force** — an attacker making 1 attempt per hour stays under `maxretry` indefinitely. Mitigate with longer `findtime` or key-only SSH auth (see `ssh-hardening.md`).
- **Distributed attacks** — if 1000 IPs each make 2 attempts, no single IP hits `maxretry`. Key-only SSH auth is the only real defense here.
- **Zero-day exploits** — Fail2ban reads logs after the fact. A successful exploit does not leave failed-auth log lines.
- **Services without log output** — if a service does not log authentication failures, Fail2ban cannot watch it.

---

## Bottom line

Fail2ban is not a substitute for key-only SSH authentication or a properly configured firewall. It is the automated response layer that handles the noise — the scanners, the credential stuffers, the bots that probe every open port on the internet.

Configure SSH key-only auth. Configure UFW with `deny incoming`. Then configure Fail2ban to watch what is left.

The recidive jail is the one most people skip. Set it up. An IP that comes back after a 28-day ban deserves a year off.

---

*Part of [sovereign-linux-tools](https://github.com/shadowbipnode/sovereign-linux-tools) — practical guides for digital sovereignty.*
