# Debian Server Security: File Integrity Monitoring (AIDE)

This repository/documentation tracks the hardening and configuration of **AIDE (Advanced Intrusion Detection Environment)** on a Debian server, focusing on eliminating false-positive noise and establishing an uncompromised baseline.

---

## Architecture & Problem Resolution

### 1. The Challenge of Volatile State Files

Out of the box, broad AIDE rules (`/etc Full`, etc.) capture volatile system paths that naturally shift during normal operations (e.g., DHCP lease updates, login session logs, audit trails, and command history). This leads to alert fatigue and renders FIM logs unactionable.

### 2. Overcoming Rule Precedence ("First Match Wins")

AIDE evaluates rules sequentially from top to bottom. The first matching rule takes precedence. 
- **The Pitfall:** Placing exclusions in custom configuration files (`05_` or `31_` prefixes) failed because Debian's base template definitions or catch-all rules further down the stack re-introduced positive monitoring for those paths.
- **The Solution:** Absolute negative override rules (`!`) must be enforced at the absolute tail of the master configuration file (`/etc/aide/aide.conf`) to ensure they intercept and halt rule evaluation before default monitoring hooks can trigger.

---

## Configuration Reference

The final, working exclusion block appended to the end of `/etc/aide/aide.conf`:

```Plaintext
# --- Override: Force Ignore Volatile Logs & State ---
!/var/log/.*
!/var/lib/faillock/.*
!/run/NetworkManager/.*
!/var/lib/NetworkManager/.*
!/home/r4k00n/\.bash_history$
!/var/spool/anacron/.*
```

---

# Operational & Automation Commands
## Re-initializing the Baseline

Whenever system rules or exclusions are modified, the database must be re-synced:

```Bash
sudo aideinit
# Answer 'Y' to overwrite aide.db.new and aide.db when prompted
```

## Running Integrity Checks

To verify system integrity against the locked baseline:

```Bash
sudo aide --check --config /etc/aide/aide.conf
```

## Automation via Systemd

To ensure integrity checks run persistently without manual intervention, use a native systemd service and timer.
1. Create the Service (`/etc/systemd/system/aide-check.service`)

```Plaintext
[Unit]
Description=AIDE File Integrity Check
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/bin/aide --check --config /etc/aide/aide.conf
# AIDE returns exit status 1 when differences are found. 
# We whitelist this so systemd treats it as a successful execution rather than a unit failure.
SuccessExitStatus=0 1
```

2. Create the Timer (`/etc/systemd/system/aide-check.timer`)

```Plaintext
[Unit]
Description=Run AIDE File Integrity Check Daily
Requires=aide-check.service

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=15m

[Install]
WantedBy=timers.target
```

3. Enable and Verify Timer

```Bash
sudo systemctl daemon-reload
sudo systemctl enable --now aide-check.timer
sudo systemctl list-timers aide-check.timer
```