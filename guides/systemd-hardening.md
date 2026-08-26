# systemd Service Hardening on Linux

Practical systemd sandboxing and privilege reduction for self-hosted Linux servers and Bitcoin/Lightning infrastructure.

The goal is simple:

> A compromised service should not automatically mean a compromised server.

systemd provides a powerful set of sandboxing features that can restrict what a daemon can see, modify, execute, or access.

But hardening is **not** copying the strictest possible configuration into every service.

Hardening means removing privileges you have verified the service does not need.

---

## 1. Before You Change Anything

Never harden a production service blindly.

First identify the unit and inspect its current configuration.

```bash
systemctl status example.service
systemctl cat example.service
systemctl show -p FragmentPath example.service
systemctl show -p DropInPaths example.service
```

Before changing anything, make sure you understand:

- which user the service runs as
- which files it reads
- which files it writes
- whether it needs network access
- whether it needs `/home`
- whether it needs `/proc` or `/sys`
- whether it creates child processes
- whether it requires Linux capabilities
- whether it needs access to devices

Do not assume.

Verify.

---

## 2. Measure the Current Security Exposure

systemd includes a useful security analyzer:

```bash
systemd-analyze security example.service
```

Analyze all services:

```bash
systemd-analyze security
```

After hardening, run the command again:

```bash
systemd-analyze security example.service
```

The score is useful for comparison, but it is **not a security certification**.

A lower number does not automatically mean the service is correctly configured.

Functionality comes first.

---

## 3. Do Not Modify Vendor Unit Files

Avoid directly editing vendor unit files under locations such as:

```text
/usr/lib/systemd/system/
/lib/systemd/system/
```

Package upgrades may replace them.

Use a systemd drop-in:

```bash
sudo systemctl edit example.service
```

This normally creates an override under:

```text
/etc/systemd/system/example.service.d/override.conf
```

Add directives under:

```ini
[Service]
```

For example:

```ini
[Service]
NoNewPrivileges=yes
```

After changing unit configuration, reload systemd when needed:

```bash
sudo systemctl daemon-reload
```

---

## 4. Start With `NoNewPrivileges`

A good first restriction for many services is:

```ini
[Service]
NoNewPrivileges=yes
```

This prevents the service and its descendants from gaining additional privileges through mechanisms such as setuid/setgid executables and file capabilities during `execve()`.

It does **not** magically make a privileged service unprivileged.

Verify:

```bash
systemctl show example.service -p NoNewPrivileges
```

Then restart:

```bash
sudo systemctl restart example.service
systemctl status example.service
journalctl -u example.service -n 50 --no-pager
```

Do not continue until the service is confirmed healthy.

---

## 5. Give the Service a Private `/tmp`

Many services do not need to share temporary files with the rest of the system.

```ini
[Service]
PrivateTmp=yes
```

This gives the service private `/tmp` and `/var/tmp` namespaces.

A service that intentionally exchanges data through these locations may break.

Test:

```bash
sudo systemctl restart example.service
systemctl status example.service
journalctl -u example.service -n 50 --no-pager
```

---

## 6. Protect the Operating System Filesystem

`ProtectSystem=` can make important parts of the filesystem read-only from the service's perspective.

A useful progression is:

```ini
ProtectSystem=yes
```

then:

```ini
ProtectSystem=full
```

and, if compatible:

```ini
ProtectSystem=strict
```

`strict` is powerful. Do not enable it blindly.

Explicitly allow required write locations:

```ini
[Service]
ProtectSystem=strict
ReadWritePaths=/var/lib/example
ReadWritePaths=/var/log/example
```

The principle is:

> Make everything read-only, then explicitly allow only the paths the service must modify.

---

## 7. Protect User Home Directories

A network daemon usually has no reason to inspect user home directories.

```ini
ProtectHome=yes
```

Before enabling this, verify whether the application stores configuration, wallets, credentials, databases, or runtime data below:

```text
/home
/root
/run/user
```

On self-hosted systems, applications are sometimes deliberately installed under a user's home directory.

Do not assume.

---

## 8. Protect Kernel Tunables

For services that do not need to modify kernel parameters:

```ini
ProtectKernelTunables=yes
```

For an ordinary application daemon, changing kernel runtime parameters should be exceptional.

---

## 9. Protect Kernel Modules

Most application services should never load or unload kernel modules.

```ini
ProtectKernelModules=yes
```

Do not apply this to software whose legitimate function requires module management.

---

## 10. Protect Control Groups

For ordinary services:

```ini
ProtectControlGroups=yes
```

Container managers and software that deliberately manages cgroups may require access.

---

## 11. Protect Kernel Logs

Where supported and appropriate:

```ini
ProtectKernelLogs=yes
```

A normal application daemon rarely needs access to the kernel log buffer.

---

## 12. Restrict Linux Capabilities

Inspect the current configuration:

```bash
systemctl show example.service -p CapabilityBoundingSet
```

systemd can restrict capabilities with:

```ini
CapabilityBoundingSet=
```

Do **not** copy a capability list from another service.

Determine exactly what the daemon requires.

Incorrect capability restrictions commonly cause services to fail at startup or lose functionality.

---

## 13. Restrict Address Families

A conventional IP service might require:

```ini
RestrictAddressFamilies=AF_INET AF_INET6
```

A service using Unix sockets may also need:

```ini
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
```

Applications using Netlink or other socket families may require additional access.

---

## 14. Restrict Namespaces

If a daemon has no reason to create Linux namespaces:

```ini
RestrictNamespaces=yes
```

Container runtimes and sandboxing software may legitimately depend on namespaces.

---

## 15. System Call Filtering

systemd supports seccomp-based system call filtering.

For example:

```ini
SystemCallFilter=@system-service
```

This can significantly reduce the kernel attack surface exposed to a compromised service.

It can also break software in subtle ways.

Treat syscall filtering as an advanced hardening step.

After enabling it:

```bash
sudo systemctl restart example.service
systemctl status example.service
journalctl -u example.service -n 100 --no-pager
```

Test the application's actual functionality, not just whether the process remains running.

---

## 16. `PrivateNetwork=yes` Is Not a Generic Hardening Switch

This directive is powerful:

```ini
PrivateNetwork=yes
```

It gives the service a private network namespace.

For many Bitcoin, Lightning, web, monitoring, Tor, database, and remote-access services, enabling this blindly will break networking.

A daemon that needs to:

- connect to Bitcoin peers
- connect to Lightning peers
- reach Tor
- query DNS
- expose an HTTP API
- connect to a database over TCP
- access external APIs

may not work with `PrivateNetwork=yes`.

Use it only when you have verified that the service does not require normal host networking.

---

## 17. Explicit Filesystem Access

For highly restricted services:

```ini
[Service]
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/example
ReadWritePaths=/var/log/example
```

You can also explicitly make paths inaccessible:

```ini
InaccessiblePaths=/media
InaccessiblePaths=/mnt
```

Or force selected locations read-only:

```ini
ReadOnlyPaths=/etc/example
```

Filesystem restrictions can directly limit what a compromised process can alter.

---

## 18. A Conservative Starting Profile

For many conventional daemons, a **starting point for testing** might be:

```ini
[Service]
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=full
ProtectHome=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
ProtectKernelLogs=yes
```

This is **not a universal configuration**.

Do not paste it into production and restart a critical service without understanding its requirements.

Add one or a small number of restrictions at a time.

Test. Observe. Continue.

---

## 19. Bitcoin and Lightning Services Need Special Care

Bitcoin and Lightning software often has requirements generic hardening examples ignore.

A Bitcoin daemon may need:

- persistent blockchain storage
- peer networking
- RPC sockets
- Tor connectivity
- configuration files
- cookie authentication files

A Lightning daemon may need:

- wallet storage
- channel database access
- macaroon files
- TLS keys
- Bitcoin RPC access
- peer networking
- Tor
- backup files

For example, this would be dangerous to copy blindly:

```ini
ProtectSystem=strict
ProtectHome=yes
PrivateNetwork=yes
```

For node infrastructure, first map:

```text
configuration
data
wallet
logs
backups
sockets
network dependencies
```

Then restrict everything else.

---

## 20. Verify Before Restarting

Inspect the resulting configuration:

```bash
systemctl cat example.service
```

Verify the unit:

```bash
systemd-analyze verify example.service
```

Configuration verification cannot prove that the application will function correctly under the new sandbox.

Only a functional test can do that.

---

## 21. Restart and Watch the Logs

After every meaningful hardening change:

```bash
sudo systemctl restart example.service
systemctl status example.service
journalctl -u example.service -n 100 --no-pager
```

For live testing:

```bash
journalctl -fu example.service
```

Do not consider the change successful merely because systemd reports:

```text
active (running)
```

Test what the service actually does.

For Bitcoin or Lightning infrastructure that may include:

- RPC
- peer connectivity
- Tor connectivity
- wallet access
- database writes
- API access
- backups
- dependent applications

---

## 22. Compare the Security Score

After a successful change:

```bash
systemd-analyze security example.service
```

Do not optimize for the score.

Optimize for the smallest privileges compatible with correct operation.

---

## 23. Roll Back a Broken Override

Always know the rollback procedure **before** restarting a production daemon.

Inspect:

```bash
systemctl cat example.service
systemctl show example.service -p DropInPaths
```

Edit:

```bash
sudo systemctl edit example.service
```

Remove or correct the directive that caused the failure.

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart example.service
systemctl status example.service
journalctl -u example.service -n 100 --no-pager
```

Do not delete unrelated drop-ins simply because one hardening directive caused a problem.

---

## 24. Suggested Hardening Workflow

```text
1. Inspect the service
2. Record the current configuration
3. Run systemd-analyze security
4. Identify required files and network access
5. Create a drop-in with systemctl edit
6. Add one restriction
7. Restart
8. Inspect logs
9. Test real application functionality
10. Measure again
11. Add the next restriction
12. Stop when additional restrictions no longer make operational sense
```

The objective is not:

```text
0.0 PERFECT
```

The objective is:

```text
minimum required privilege
+
fully functional service
+
known rollback procedure
```

---

## 25. Quick Audit Commands

```bash
systemctl --type=service --state=running
systemd-analyze security
systemctl cat example.service
systemctl show example.service -p User -p Group
```

Inspect important sandbox properties:

```bash
systemctl show example.service \
  -p NoNewPrivileges \
  -p PrivateTmp \
  -p ProtectSystem \
  -p ProtectHome \
  -p ProtectKernelTunables \
  -p ProtectKernelModules \
  -p ProtectControlGroups \
  -p ProtectKernelLogs \
  -p RestrictAddressFamilies
```

Inspect recent warnings:

```bash
journalctl -u example.service -p warning --no-pager
```

---

## 26. Final Checklist

- [ ] Original unit inspected
- [ ] Existing drop-ins identified
- [ ] Service user/group understood
- [ ] Required write paths identified
- [ ] Required network access identified
- [ ] Initial `systemd-analyze security` recorded
- [ ] Changes made through a drop-in
- [ ] `NoNewPrivileges` tested
- [ ] Filesystem protections tested
- [ ] Kernel protections tested where appropriate
- [ ] Capabilities reviewed
- [ ] Address families reviewed
- [ ] Namespace restrictions considered
- [ ] Syscall filtering considered
- [ ] Service restarted successfully
- [ ] Logs checked
- [ ] Real application functionality tested
- [ ] Final security analysis compared
- [ ] Rollback procedure known

---

## The Principle

systemd sandboxing is powerful because it assumes something important:

**software can fail.**

A firewall protects what can reach a service.

SSH hardening protects administrative access.

fail2ban reduces repeated hostile access attempts.

systemd sandboxing addresses a different question:

> What happens after the daemon itself is compromised?

The answer should not be:

> It can access everything its host can access.

Give every service only the privileges it actually needs.

Nothing more.

---

## References

- systemd documentation: `systemd.exec(5)`
- systemd documentation: `systemd-analyze(1)`
- systemd documentation: `systemctl(1)`
- Linux kernel documentation: `no_new_privs`

Read the locally installed documentation:

```bash
man systemd.exec
man systemd-analyze
man systemctl
```

---

*Verify everything. Trust nothing you haven't run yourself.*
