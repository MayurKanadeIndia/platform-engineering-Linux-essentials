# 📋 Linux System Logging

> **Learning Path:** Linux Fundamentals → Infrastructure Engineering → Cloud Engineering  
> **Level:** Beginner to Intermediate  
> **Goal:** Understand how Linux records, categorizes, stores, and manages every event that happens on a system — and how to use logs to debug, monitor, and maintain production servers.

---

## 📚 Table of Contents

1. [What We Will Learn](#what-we-will-learn)
2. [Why Logs Exist](#why-logs-exist)
3. [The Syslog Standard](#the-syslog-standard)
4. [Facilities — WHO Sent the Message](#facilities--who-sent-the-message)
5. [Severities — HOW Serious Is It](#severities--how-serious-is-it)
6. [Syslog Servers](#syslog-servers)
7. [Logging Rules — Selector and Action Fields](#logging-rules--selector-and-action-fields)
8. [Where Logs Are Stored](#where-logs-are-stored)
9. [The logger Command](#the-logger-command)
10. [Log Rotation with logrotate](#log-rotation-with-logrotate)
11. [The Complete Architecture](#the-complete-architecture)
12. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
13. [Interview Questions](#interview-questions)
14. [Common Beginner Mistakes](#common-beginner-mistakes)

---

## What We Will Learn

- What the syslog standard is and why it exists
- Facilities — which part of the system sent a message
- Severities — how serious a log message is (levels 0–7)
- How syslog servers (rsyslog, syslog-ng) process messages
- Logging rules — the selector and action field system
- Where different log files are stored in `/var/log/`
- How to generate your own log messages with `logger`
- How to automatically rotate, compress, and clean up old logs with `logrotate`

---

## Why Logs Exist

Logs are your system's **permanent diary**. Every event, action, warning, and error is written down automatically so engineers can:

- Diagnose what went wrong and exactly when
- Monitor system health in real time
- Audit who did what and from where (security)
- Comply with regulations that require audit trails
- Reconstruct a timeline after an incident

```
Without logs:
  Server crashes at 3 AM
  Engineer wakes up
  "Why did it crash?" → No idea. Just guess and hope. 😱

With logs:
  Server crashes at 3 AM
  Engineer reads /var/log/syslog
  "Disk filled up at 02:47 AM → rsyslog stopped → app crashed at 02:51 AM"
  Root cause found in 60 seconds. Fix applied. ✅
```

> In production at any major tech company, logs are the first thing an engineer checks when something goes wrong. If you cannot read logs, you cannot do infrastructure work.

---

## The Syslog Standard

Linux uses the **syslog standard** for all system message logging. It provides a unified, centrally-controlled way to handle logs across the entire operating system.

### What the Standard Defines

Every syslog message is labeled with exactly two things:

```
┌───────────────────┬────────────────────┬──────────────────────────────┐
│   FACILITY        │    SEVERITY        │   MESSAGE                    │
│  (who sent it)    │  (how serious)     │  (what happened)             │
├───────────────────┼────────────────────┼──────────────────────────────┤
│  kern             │  error             │  "disk read error on sda"    │
│  mail             │  info              │  "email delivered to john"   │
│  auth             │  warning           │  "failed login from x.x.x.x" │
└───────────────────┴────────────────────┴──────────────────────────────┘
```

### Why a Standard Matters

Without a standard, every program would log in its own format — making it impossible to centrally collect, filter, and manage logs across thousands of services. The syslog standard means one configuration file controls where every message from every program goes.

---

## Facilities — WHO Sent the Message

A **facility** identifies which part of the system or which application generated the log message.

### Facility Reference Table

| Facility        | Number | Source                                        |
| --------------- | ------ | --------------------------------------------- |
| `kern`          | 0      | Linux Kernel                                  |
| `user`          | 1      | User-level programs                           |
| `mail`          | 2      | Mail/email system                             |
| `daemon`        | 3      | Background system daemons                     |
| `auth`          | 4      | Security and authorization (login, sudo, SSH) |
| `syslog`        | 5      | syslogd internal messages                     |
| `lpr`           | 6      | Printer subsystem                             |
| `news`          | 7      | Network news subsystem                        |
| `uucp`          | 8      | UUCP subsystem                                |
| `cron`          | 9      | Cron scheduled task daemon                    |
| `authpriv`      | 10     | Private authentication messages               |
| `ftp`           | 11     | FTP daemon                                    |
| `local0–local7` | 16–23  | **Reserved for your custom applications**     |

### The local0–local7 Facilities

`local0` through `local7` are specifically reserved for **custom applications and scripts** that you build yourself. This is how you integrate your own app into the Linux logging system.

```bash
# Example: Your app sends logs as local0
logger -p local0.info "MyApp: Server started on port 8080"
logger -p local0.error "MyApp: Database connection refused"

# Configure rsyslog to route local0 messages to its own file
# In /etc/rsyslog.d/myapp.conf:
# local0.*  /var/log/myapp.log
```

---

## Severities — HOW Serious Is It

Every syslog message has a **severity level** from 0 (most critical) to 7 (least critical). Lower number = more serious.

### Severity Level Reference

| Code | Severity  | Keyword            | Meaning                               | When You See It                                 |
| ---- | --------- | ------------------ | ------------------------------------- | ----------------------------------------------- |
| 0    | Emergency | `emerg` / `panic`  | System is completely unusable         | Kernel panic, complete system failure           |
| 1    | Alert     | `alert`            | Must act immediately                  | Data corruption detected, critical process died |
| 2    | Critical  | `crit`             | Critical hardware or software failure | Primary disk failed, backup device lost         |
| 3    | Error     | `err` / `error`    | Error conditions — something failed   | Service failed to start, file not found         |
| 4    | Warning   | `warning` / `warn` | Not broken yet, but watch out         | Disk 85% full, high memory usage                |
| 5    | Notice    | `notice`           | Normal but significant                | Service restarted, config reloaded              |
| 6    | Info      | `info`             | Normal informational messages         | User logged in, request served                  |
| 7    | Debug     | `debug`            | Detailed debugging information        | Variable values, function traces                |

### Severity Filtering Behavior

When you specify a severity in a logging rule, it matches that level **AND ALL HIGHER SEVERITY LEVELS** (lower numbers).

```
Rule: mail.warn  →  /var/log/mail.warn

This captures:
  ✅ warning  (level 4)
  ✅ error    (level 3)
  ✅ critical (level 2)
  ✅ alert    (level 1)
  ✅ emerg    (level 0)
  ❌ notice   (level 5) — NOT captured
  ❌ info     (level 6) — NOT captured
  ❌ debug    (level 7) — NOT captured
```

### Production Monitoring Strategy

```
Levels 0–2 (emerg, alert, crit):   → Page the on-call engineer NOW
Level 3 (error):                   → Create a ticket, investigate today
Level 4 (warning):                 → Review in daily standup
Level 5 (notice):                  → Review weekly
Levels 6–7 (info, debug):          → Available for manual investigation only
```

> In production environments, monitoring systems (like Google Cloud Monitoring, Datadog, PagerDuty) are configured to alert engineers when severity level 3 (error) or above appears in logs.

---

## Syslog Servers

A **syslog server** is a daemon (background process) that:

- Receives log messages from all parts of the system
- Reads the configured rules
- Routes each message to the correct destination (file, remote server, console)

### The Three Syslog Implementations

| Daemon      | Description                  | Status                               |
| ----------- | ---------------------------- | ------------------------------------ |
| `syslogd`   | The original syslog daemon   | Obsolete — replaced by rsyslog       |
| `rsyslog`   | Reliable and Extended syslog | **Standard on Ubuntu, Debian, RHEL** |
| `syslog-ng` | Syslog Next Generation       | Used in enterprise environments      |

### rsyslog — The Modern Standard

```bash
# Check rsyslog status
sudo systemctl status rsyslog

# Start rsyslog
sudo systemctl start rsyslog

# Reload config without restart (apply rule changes)
sudo systemctl reload rsyslog

# View rsyslog configuration
cat /etc/rsyslog.conf
```

### Configuration File Structure

```
Main config:           /etc/rsyslog.conf
Additional configs:    /etc/rsyslog.d/*.conf   (included automatically)
```

The `$IncludeConfig` directive in `rsyslog.conf` tells rsyslog to also load all `.conf` files in `/etc/rsyslog.d/`:

```bash
# Inside /etc/rsyslog.conf:
$IncludeConfig /etc/rsyslog.d/*.conf

# This means ALL of these are automatically loaded:
/etc/rsyslog.d/50-default.conf    ← Ubuntu's default logging rules
/etc/rsyslog.d/myapp.conf         ← Your custom app rules (drop here!)
```

> **Best practice:** Never edit `/etc/rsyslog.conf` directly. Drop your custom rules as a new `.conf` file in `/etc/rsyslog.d/`. This keeps the main config clean and makes your changes easy to identify and remove.

---

## Logging Rules — Selector and Action Fields

A logging rule tells rsyslog: **"If a message matches THIS pattern, do THIS with it."**

Every rule consists of exactly two fields separated by one or more spaces or tabs:

```
SELECTOR FIELD          ACTION FIELD
──────────────────────────────────────────────────────
mail.*               /var/log/mail.log
auth,authpriv.*      /var/log/auth.log
*.err                /var/log/errors.log
```

### Field 1: The Selector Field

Format: `FACILITY.SEVERITY`

```
SELECTOR SYNTAX EXAMPLES:
┌──────────────────────────────┬───────────────────────────────────────────────┐
│ Selector                     │ Meaning                                       │
├──────────────────────────────┼───────────────────────────────────────────────┤
│ mail.info                    │ mail facility, info severity and above         │
│ mail.*                       │ mail facility, ALL severity levels             │
│ mail.warn                    │ mail facility, warn and above (warn/err/crit…) │
│ mail.none                    │ mail facility — EXCLUDE all mail messages      │
│ *.err                        │ ALL facilities, error severity and above       │
│ *.*                          │ Literally everything                           │
│ auth,authpriv.*              │ auth AND authpriv facilities, all severities   │
│ *.info;mail.none             │ All info+ messages EXCEPT mail                 │
│ *.info;mail.none;cron.none   │ All info+ EXCEPT mail and cron                │
└──────────────────────────────┴───────────────────────────────────────────────┘
```

**Wildcard and special keywords:**

| Symbol          | Meaning                                           |
| --------------- | ------------------------------------------------- |
| `*` on facility | All facilities                                    |
| `*` on severity | All severity levels                               |
| `.none`         | Exclude this facility entirely                    |
| `;`             | Separate multiple selectors in one rule           |
| `,`             | Match multiple facilities (e.g., `auth,authpriv`) |

### Field 2: The Action Field

The action field defines what rsyslog **does with matched messages:**

```
ACTION EXAMPLES:
┌───────────────────────────────┬──────────────────────────────────────────┐
│ Action                        │ Meaning                                  │
├───────────────────────────────┼──────────────────────────────────────────┤
│ /var/log/mail.log             │ Write to local file (with disk sync)     │
│ -/var/log/mail.info           │ Write to local file (cached, no sync)    │
│ @192.168.1.100                │ Forward to remote syslog server via UDP  │
│ @@192.168.1.100               │ Forward to remote syslog server via TCP  │
│ /dev/console                  │ Write to system console                  │
│ *                             │ Broadcast to all logged-in users         │
└───────────────────────────────┴──────────────────────────────────────────┘
```

### The Minus Sign (-) — Performance Caching

**IMPORTANT:** The `-` prefix before a file path enables **write caching** (asynchronous I/O).

```
WITHOUT minus (synchronous — safe):
  mail.err    /var/log/mail.err

  Behavior: Every message → immediately flushed to disk
  Disk writes: One per message
  Performance: Slower (high disk I/O)
  Safety: No data loss even on sudden crash ✅

WITH minus (asynchronous — fast):
  mail.info  -/var/log/mail.info

  Behavior: Messages buffered in memory → flushed periodically
  Disk writes: Many messages per flush (batched)
  Performance: Much faster (much lower disk I/O) ✅
  Safety: Messages in buffer lost if server crashes suddenly ⚠️
```

**Usage guidelines:**

```
Critical severity (0–2):  NEVER use minus → /var/log/critical.log
Error severity (3):       Avoid minus    → /var/log/errors.log
Warning and below (4–7):  Use minus      → -/var/log/app.log
```

### Complete Logging Rules Examples

**Ubuntu/Debian default (`/etc/rsyslog.d/50-default.conf`):**

```bash
# Auth and authpriv → dedicated security log (no caching — security critical)
auth,authpriv.*                  /var/log/auth.log

# Everything except auth/authpriv → general syslog (cached)
*.*;auth,authpriv.none           -/var/log/syslog

# Kernel messages (cached)
kern.*                           -/var/log/kern.log

# Mail info and above (cached)
mail.info                        -/var/log/mail.log

# Mail warnings (cached)
mail.warn                        -/var/log/mail.warn

# Mail errors (NOT cached — important!)
mail.err                         /var/log/mail.err
```

**Red Hat / CentOS default:**

```bash
# All info+ except mail, authpriv, cron → /var/log/messages
*.info;mail.none;authpriv.none;cron.none    /var/log/messages

# Auth messages → /var/log/secure
authpriv.*                                  /var/log/secure

# Mail messages → /var/log/maillog
mail.*                                      -/var/log/maillog

# Cron messages → /var/log/cron
cron.*                                      /var/log/cron
```

### Adding Your Own Custom Rule

```bash
# Create a new rule file for your application
sudo nano /etc/rsyslog.d/myapp.conf

# Add your rule:
local0.*    /var/log/myapp.log

# Reload rsyslog to apply the new rule
sudo systemctl reload rsyslog

# Test it
logger -p local0.info "Test message from my app"
tail -3 /var/log/myapp.log
```

---

## Where Logs Are Stored

All system log files are stored under `/var/log/`.

### Key Log Files

```
/var/log/
├── syslog              ← General messages — Ubuntu/Debian (MAIN LOG)
├── messages            ← General messages — RHEL/CentOS (MAIN LOG)
├── auth.log            ← Logins, sudo, SSH — Ubuntu/Debian
├── secure              ← Logins, sudo, SSH — RHEL/CentOS
├── kern.log            ← Kernel messages
├── mail.log            ← Email system messages
├── cron.log            ← Scheduled task (cron) messages
├── boot.log            ← Messages from the boot process
├── dmesg               ← Kernel ring buffer (hardware detection)
├── dpkg.log            ← Package install/remove history (Debian)
├── yum.log             ← Package history (RHEL/CentOS)
├── wtmp                ← Login history (binary — read with 'last')
├── btmp                ← Failed login attempts (binary — read with 'lastb')
├── faillog             ← Failed login count per user
└── apache2/ or nginx/  ← Web server logs (if installed)
    ├── access.log      ← Every HTTP request received
    └── error.log       ← Web server errors
```

### Commands to Read Log Files

```bash
# View entire file
cat /var/log/syslog

# View last N lines (most recent entries)
tail -20 /var/log/syslog

# Watch file LIVE as new messages arrive (most used in production)
tail -f /var/log/syslog
tail -f /var/log/auth.log

# Search for specific text
grep "error" /var/log/syslog
grep "Failed password" /var/log/auth.log
grep "sshd" /var/log/auth.log

# Search case-insensitively
grep -i "warning" /var/log/syslog

# View login history (reads /var/log/wtmp)
last

# View failed login attempts (reads /var/log/btmp)
sudo lastb
```

---

## The logger Command

The `logger` command allows **any user or script** to write messages directly into the syslog system from the terminal or from within a shell script.

### Syntax

```bash
logger [options] "your message"

Options:
  -p FACILITY.SEVERITY   Specify facility and severity level
                         (defaults to user.notice if not specified)
  -t TAG                 Add an identifying tag to the message
```

### Examples

```bash
# Simple message with default facility.severity (user.notice)
logger "Server health check passed"

# Specify facility and severity
logger -p mail.info "Mail queue processed successfully"
logger -p kern.warning "Unusual kernel activity detected"

# Add a tag to identify the source
logger -t MyWebApp "Application started on port 8080"

# Both options together
logger -p local0.error -t BackupScript "Backup job failed at /mnt/data"

# Verify the message was logged
sudo tail -5 /var/log/syslog
```

### Using logger in Shell Scripts

```bash
#!/bin/bash
# backup.sh — production backup script with proper logging

APP_NAME="DailyBackup"
SOURCE="/var/data"
DEST="/backup/daily"

logger -t "$APP_NAME" -p local0.info "Backup started: $SOURCE → $DEST"

# Perform backup
rsync -az "$SOURCE" "$DEST"
EXIT_CODE=$?

if [ $EXIT_CODE -eq 0 ]; then
    logger -t "$APP_NAME" -p local0.info "Backup completed successfully"
else
    logger -t "$APP_NAME" -p local0.error "Backup FAILED with exit code $EXIT_CODE"
fi
```

> Messages generated by `logger` appear in `/var/log/syslog` (or wherever your rsyslog rules route them). They are treated exactly the same as messages from any other facility.

---

## Log Rotation with logrotate

Without log rotation, log files grow indefinitely until they fill the disk and crash the server. `logrotate` is the standard tool for **automatically rotating, compressing, and deleting old log files** on a schedule.

### What Log Rotation Does

```
Before rotation:
  /var/log/syslog         ← Active file (10 GB and growing)

After weekly rotation:
  /var/log/syslog         ← NEW empty file (rsyslog writes here now)
  /var/log/syslog.1       ← Last week's log (uncompressed)
  /var/log/syslog.2.gz    ← 2 weeks ago (compressed with gzip)
  /var/log/syslog.3.gz    ← 3 weeks ago (compressed)
  /var/log/syslog.4.gz    ← 4 weeks ago (compressed)
                          ← 5 weeks ago: AUTOMATICALLY DELETED
```

### Configuration Files

```
Main config:          /etc/logrotate.conf
Application configs:  /etc/logrotate.d/   (one file per application)
```

`/etc/logrotate.conf` sets **global defaults** and includes all files in `/etc/logrotate.d/`.

### Global Default Config (`/etc/logrotate.conf`)

```bash
weekly          # Rotate logs every week globally
rotate 4        # Keep last 4 rotations (4 weeks of history)
create          # Create a fresh new empty log file after rotating
compress        # Compress old log files with gzip
include /etc/logrotate.d   # Also load all app-specific configs
```

### Application-Specific Config Example

```bash
# Manages /var/log/debug and /var/log/messages:
/var/log/debug
/var/log/messages
{
    rotate 4          # Keep 4 old copies
    weekly            # Rotate every 7 days
    missingok         # Don't error if file doesn't exist
    notifempty        # Skip rotation if file is empty
    compress          # Compress old files (.gz)
    sharedscripts     # Run postrotate once for all matched files
    postrotate
        reload rsyslog >/dev/null 2>&1 || true
                      # Tell rsyslog to open the new log file
    endscript
}
```

### logrotate Directives Reference

| Directive       | Meaning                                                                |
| --------------- | ---------------------------------------------------------------------- |
| `rotate N`      | Keep N old copies. When N+1th rotation happens, the oldest is deleted. |
| `daily`         | Rotate every day                                                       |
| `weekly`        | Rotate every week                                                      |
| `monthly`       | Rotate every month                                                     |
| `size 100M`     | Rotate when file reaches 100MB (regardless of time)                    |
| `missingok`     | Do not produce an error if the log file is missing                     |
| `notifempty`    | Do not rotate if the log file is empty                                 |
| `compress`      | Compress old log files using gzip (creates `.gz` files)                |
| `nocompress`    | Do not compress old log files                                          |
| `create`        | Create a new empty log file after rotation                             |
| `sharedscripts` | Run `postrotate` script once for all files, not once per file          |
| `postrotate`    | Begins a script block to run immediately after rotation                |
| `endscript`     | Ends the script block                                                  |
| `dateext`       | Append date to filename instead of number (e.g., syslog-20260218)      |

### Why Reload rsyslog After Rotation?

```
Problem:
  rsyslog has /var/log/syslog open and is actively writing to it.
  logrotate renames it to /var/log/syslog.1.
  rsyslog still has the old file descriptor open.
  rsyslog keeps writing to syslog.1 (the renamed file)!
  The new /var/log/syslog file exists but nothing writes to it.

Solution: postrotate → reload rsyslog
  rsyslog closes the old file descriptor.
  rsyslog opens the new /var/log/syslog.
  Writing now goes to the correct fresh file. ✅
```

### Testing logrotate

```bash
# Force a rotation right now and show verbose output (dry run equivalent)
sudo logrotate -fv /etc/logrotate.conf

# Force rotation of a specific application's config
sudo logrotate -fv /etc/logrotate.d/rsyslog

# Options:
#  -f = force rotation even if it's not time yet
#  -v = verbose — show detailed output of what it's doing
```

---

## The Complete Architecture

```
SYSTEM COMPONENTS generate messages:
  Linux Kernel  → kern.error   "disk read error"
  SSH daemon    → auth.info    "Accepted publickey for john"
  Nginx         → daemon.info  "worker process started"
  Your script   → local0.info  "Backup completed"
                        │
                        │ All sent to syslog standard
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      RSYSLOG DAEMON                             │
│  Reads: /etc/rsyslog.conf + /etc/rsyslog.d/*.conf               │
│                                                                  │
│  Applies rules (SELECTOR → ACTION):                             │
│    auth,authpriv.*      → /var/log/auth.log   (no cache)        │
│    *.*;auth.none        → -/var/log/syslog    (cached)          │
│    kern.*               → -/var/log/kern.log  (cached)          │
│    mail.err             → /var/log/mail.err   (no cache)        │
│    local0.*             → /var/log/myapp.log  (custom rule)     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Routes messages to correct files
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                      /var/log/                                  │
│  auth.log    syslog    kern.log    mail.log    myapp.log    ...  │
│  (Files grow continuously as system events occur)               │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Files grow too large over time
                           ▼ logrotate runs on schedule (cron)
┌─────────────────────────────────────────────────────────────────┐
│                    LOGROTATE                                     │
│  Reads: /etc/logrotate.conf + /etc/logrotate.d/*                │
│                                                                  │
│  For each log file:                                             │
│    Renames current log  → syslog.1                              │
│    Compresses older log → syslog.2.gz                           │
│    Deletes logs beyond  → rotate count (e.g., 4 weeks)          │
│    Creates fresh file   → new /var/log/syslog                   │
│    Reloads rsyslog      → write to new file                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Cheat Sheet

```bash
# ── VIEWING LOG FILES ─────────────────────────────────────────
tail -f /var/log/syslog              # Watch general log live
tail -f /var/log/auth.log            # Watch security log live
tail -100 /var/log/syslog            # Last 100 lines
grep "error" /var/log/syslog         # Search for errors
grep -i "failed" /var/log/auth.log   # Case-insensitive search
grep "sshd" /var/log/auth.log        # Filter by program name
cat /var/log/syslog | less           # Browse with pagination

# ── SYSTEMD JOURNAL (modern log viewer) ───────────────────────
journalctl                           # All logs
journalctl -b                        # Logs since last boot
journalctl -b -1                     # Logs from previous boot
journalctl -f                        # Watch live
journalctl -u nginx                  # Logs for specific service
journalctl -p err                    # Only error level and above
journalctl --since "1 hour ago"      # Last 1 hour of logs
journalctl --since "2026-01-01"      # Since specific date

# ── RSYSLOG ───────────────────────────────────────────────────
sudo systemctl status rsyslog        # Check rsyslog is running
sudo systemctl reload rsyslog        # Reload config (after rule changes)
sudo systemctl restart rsyslog       # Full restart

# ── GENERATING TEST LOG MESSAGES ─────────────────────────────
logger "Test message"                               # Default: user.notice
logger -p mail.info "Mail test"                     # Specific facility.severity
logger -t MyApp "Application event"                 # With tag
logger -p local0.error -t MyApp "Error occurred"   # Both options

# ── LOGROTATE ─────────────────────────────────────────────────
sudo logrotate -fv /etc/logrotate.conf              # Force rotate all logs
sudo logrotate -fv /etc/logrotate.d/rsyslog         # Rotate specific config
cat /etc/logrotate.conf                             # View global config
ls /etc/logrotate.d/                                # View app-specific configs

# ── LOGIN HISTORY ─────────────────────────────────────────────
last                                 # Login history (reads /var/log/wtmp)
sudo lastb                           # Failed login attempts (reads /var/log/btmp)
sudo faillog                         # Failed login counts per user
```

---

## Interview Questions

**Beginner Level:**

- What is syslog and why does Linux use a logging standard?
- What is the difference between a Facility and a Severity?
- Name three severity levels from most to least critical.
- What does `tail -f /var/log/syslog` do?
- What is the purpose of logrotate?

**Intermediate Level:**

- Explain the two fields of a logging rule in rsyslog. Give an example.
- What does the `-` (minus sign) prefix before a log file path mean in rsyslog rules?
- Why do you need to reload rsyslog after logrotate runs?
- What is the difference between `/var/log/syslog` and `/var/log/auth.log`?
- Write a logging rule that captures all messages from `local0` facility and writes them to `/var/log/myapp.log`.

**Senior Level:**

- A production server's disk filled up at 3 AM and the application crashed. Walk me through exactly which log files you would check and what commands you would run to diagnose the issue.
- How would you configure rsyslog to forward all `error` and above messages from your application to a central log server?
- A new microservice needs its own dedicated log file. Walk me through configuring rsyslog and logrotate for it from scratch.
- What is the difference between `rsyslog` and `journald`? When would you use each?
- Your log rotation is not working correctly — rsyslog keeps writing to the old renamed file after rotation. Why does this happen and how do you fix it?

---

## Common Beginner Mistakes

**1. Not knowing which log file to check**
Different distributions store logs differently. Ubuntu uses `/var/log/syslog` and `/var/log/auth.log`. RHEL/CentOS uses `/var/log/messages` and `/var/log/secure`. Always verify the correct file for your distribution.

**2. Confusing severity level numbering direction**
Level 0 is the MOST critical (Emergency), Level 7 is the LEAST critical (Debug). Beginners often assume higher number = more serious. It is the opposite.

**3. Forgetting that severity rules are inclusive upward**
`mail.warn` does not mean ONLY warnings. It means warnings AND everything more critical (error, critical, alert, emergency). To match ONLY one severity, use more specific rsyslog filter syntax.

**4. Editing `/etc/rsyslog.conf` directly instead of using `/etc/rsyslog.d/`**
Drop custom rules in `/etc/rsyslog.d/myapp.conf` — never edit the main file. System package updates can overwrite `/etc/rsyslog.conf` and wipe your changes.

**5. Forgetting to reload rsyslog after adding a new rule**
Adding a file to `/etc/rsyslog.d/` does nothing until you run `sudo systemctl reload rsyslog`. The new rule is not active until rsyslog re-reads its configuration.

**6. Using `-` (caching) for critical log files**
Never use the minus prefix for critical security or error logs. If the server crashes before the cache flushes, those messages are permanently lost. You cannot investigate an incident you have no record of.

**7. Not rotating custom application logs**
Engineers often configure rsyslog to write their app logs to `/var/log/myapp.log` but forget to add a corresponding `/etc/logrotate.d/myapp` config. Six months later, the file is 50GB and the disk is full.

**8. Attempting to read `/var/log/wtmp` or `/var/log/btmp` directly with cat**
These are binary files. `cat wtmp` produces garbage. Use `last` (for wtmp) and `sudo lastb` (for btmp) to read them properly.

---

## Summary

```
Syslog Standard  → Unified format for all Linux log messages
Facility         → WHO sent the message (kern, mail, auth, local0–7)
Severity         → HOW serious (0=Emergency, 7=Debug — lower = worse)
rsyslog          → The daemon that receives, routes, and stores messages
Logging Rule     → SELECTOR_FIELD  ACTION_FIELD (two parts separated by space)
Selector Field   → FACILITY.SEVERITY — what messages to match
Action Field     → Where to send matched messages (file, remote server)
Minus sign (-)   → Enable write caching for performance (use with care)
/var/log/        → Directory where all log files are stored
/var/log/syslog  → Main log file on Ubuntu/Debian
/var/log/messages→ Main log file on RHEL/CentOS
/var/log/auth.log→ Security and authentication events
logger           → Command to write your own messages into syslog
logrotate        → Tool to rotate, compress, and delete old log files
/etc/logrotate.conf    → Global logrotate configuration
/etc/logrotate.d/      → Per-application logrotate configurations
postrotate       → Script block that runs after rotation (used to reload rsyslog)
```

---

# Summary

![alt text](images/Linux_system_log.png)
