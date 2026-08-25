# 🔐 MJ-IPguard — SSH Log Monitoring, Alerting & Banning System v2.5

> Lightweight SSH attack detection, automated firewall blocking, and email alerting — no SIEM required.

---

## Problem Statement

Small and mid-sized Linux servers often lack real-time SSH attack detection without deploying complex SIEM systems. MJ-IPguard provides a lightweight alternative that:

- Detects brute-force attacks in real time by tailing `/var/log/auth.log`
- Blocks malicious IPs automatically via `firewalld` rich rules
- Generates structured logs and JSON reports for analysis
- Sends email alerts every 500 detected attacks

---

## 📋 Table of Contents

- [Overview](#overview)
- [🎯 Key Features](#-key-features)
- [⚙️ How It Works — Data Flow](#️-how-it-works--data-flow)
- [🏗️ Architecture & Directory Structure](#️-architecture--directory-structure)
- [⚡ Quick Start](#-quick-start)
- [🔧 Installation & Setup](#-installation--setup)
- [⚙️ Configuration](#️-configuration)
- [📊 Usage](#-usage)
- [📁 Output Files](#-output-files)
- [🔍 Visualization](#-visualization)
- [🛡️ Security Features](#️-security-features)
- [📈 Market Relevance — Why This Matters](#-market-relevance--why-this-matters)
- [🆚 MJ-IPguard vs Alternatives](#-mj-ipguard-vs-alternatives)
- [🎯 Who Should Use This](#-who-should-use-this)
- [🔮 Possible Future Enhancements](#-possible-future-enhancements)
- [🐛 Troubleshooting](#-troubleshooting)
- [📝 License](#-license)
- [🤔 FAQ: Why Not Just Whitelist IPs in firewalld?](#-faq-why-not-just-whitelist-ips-in-firewalld)


---

## Overview

MJ-IPguard is a Python-based security tool that:

- **Monitors** `/var/log/auth.log` in real time using tail-follow I/O
- **Detects** brute-force attacks, lockout events, and PAM failures via 5 regex patterns
- **Blocks** attacking IPs automatically using `firewall-cmd` rich rules
- **Alerts** via email (mutt first, SMTP fallback) every 500 cumulative attacks
- **Reports** structured threat data to `threat_ip.log` and `threat_ip.json`
- **Visualizes** attack counts via a live matplotlib chart or a Flask web dashboard
- **Protects** your own IP before applying firewall rules using `protect-my-ip.sh`
- **Rate-limits** SSH connections to 10/minute during setup

---

## 🎯 Key Features

### 🔴 Real-Time Threat Detection (`log_analyser.py`)

Monitors `/var/log/auth.log` continuously and matches 5 regex patterns:

| Pattern | Category |
|---|---|
| `Failed password for [user] from [IP]` | `brute_force` |
| `Failed password for [user] from ::1` | `brute_force` (localhost) |
| `error: maximum authentication attempts exceeded` | `lockout` |
| `authentication failure; rhost=[IP]` | `pam_failure` |
| `pam_unix(sshd:auth): authentication failure; rhost=` | `lockout` |

- Tail-follow I/O — only reads new lines, not the whole file
- Extracts and saves attacker IPs to `ips_detected.txt`
- Keeps last 10 log lines per attack category for forensic review

### 📧 Email Alerting (`email_handler.py` + `mutt.sh`)

- Triggers every **500 cumulative attacks** detected
- Tries `mutt` first — sends `threat_ip.log` as both body and attachment
- Falls back to SMTP (Gmail) if mutt is unavailable
- Credentials loaded from `.env` file — nothing hardcoded
- After sending, `threat_ip.log` and `threat_ip.json` are cleared for the next cycle

### 🔥 Automated Firewall Blocking (`firewall_auto_ip_blocker.py`)

- Continuously monitors `ips_detected.txt` for new IPs (polls every 3 seconds, file-size based)
- Validates each IP against a strict IPv4 regex before acting
- Adds a `firewall-cmd` drop rule per IP:
  ```bash
  firewall-cmd --add-rich-rule 'rule family="ipv4" source address="<IP>" drop'
  ```
- Uses a `set` to skip already-processed IPs (O(1) deduplication)
- Logs every action (success/failure/invalid) with timestamps to `firewall_rules.log`

### 🔧 Firewall Auto-Setup (`firewall-auto-setup.py`)

- Checks if the user is root
- Verifies `firewalld` is installed
- Enables and starts `firewalld` via `systemctl`
- Recursively retries activation if the service is not yet active

### 🛡️ Self-Protection (`protect-my-ip.sh`)

- Collects all local IPv4 addresses and optionally the public IP (via `curl`)
- Adds a high-priority (`-1`) permanent `accept` rich rule for each IP before any blocking rules are applied
- Prevents accidental lockout of the admin during firewall setup

### 📊 JSON & Log Reporting (`file_handler.py`)

- `threat_ip.log` — human-readable scorecard with per-category counts and last 10 logs
- `threat_ip.json` — machine-readable JSON with `total_attacks` and `attack_details` per category
- `output.log` — appended CLI activity log of every detection event
- `ips_detected.txt` — plain IP list consumed by the firewall blocker

### 📈 Visualization

Two modes available via `bash run.sh view`:

| Mode | File | Access |
|---|---|---|
| Terminal chart | `visualize_threats.py` | Desktop matplotlib window, refreshes every 5s |
| Web dashboard | `visualize_web.py` | `http://localhost:8080`, auto-refreshes every 5s |

Both read from `threat_ip.json` and display bar charts for `brute_force`, `lockout`, `pam_failure`, and `other_failed`.

---

## ⚙️ How It Works — Data Flow

Understanding the internal flow helps you debug, extend, or integrate MJ-IPguard into your own stack.

```
/var/log/auth.log
       │
       │  tail-follow (seek to EOF on start, read new lines only)
       ▼
  log_analyser.py  ──── regex match (5 patterns) ────►  stats dict (brute_force / lockout / pam_failure / other_failed)
       │                                                        │
       │  extract IP from matched line                         │  write on every match
       ▼                                                        ▼
 ips_detected.txt                                    threat_ip.log  +  threat_ip.json
       │                                                        │
       │  file-size poll every 3s                              │  every 500 total attacks
       ▼                                                        ▼
firewall_auto_ip_blocker.py                           email_handler.py
  │  validate IPv4 regex                                │  try mutt first
  │  skip duplicates (set)                              │  fallback → SMTP (Gmail)
  ▼                                                     ▼
firewall-cmd rich rule DROP                     alert email sent
  │                                             threat_ip.log + .json cleared
  ▼
firewall_rules.log  (audit trail)
```

**Two independent background processes run in parallel:**
- `main.py` — reads auth.log, detects threats, writes output files, triggers email
- `firewall_auto_ip_blocker.py` — watches `ips_detected.txt`, blocks IPs via firewalld

Both are started by `init.sh` using `nohup`, with PIDs stored in `analysis_output/` for clean shutdown.

---

## 🏗️ Architecture & Directory Structure

```
SSH_log_monitoring-alerting_and_banning_/
├── README.md
├── .gitignore
├── requirements.txt
│
├── working/                          ← All executable files
│   ├── main.py                       ← Entry point
│   ├── log_analyser.py               ← Detection engine
│   ├── email_handler.py              ← Email alerting (mutt + SMTP)
│   ├── file_handler.py               ← Output file writer
│   ├── firewall_auto_ip_blocker.py   ← IP blocking daemon
│   ├── firewall-auto-setup.py        ← Firewall setup checker
│   ├── visualize_threats.py          ← Matplotlib terminal chart
│   ├── visualize_web.py              ← Flask web dashboard
│   ├── init.sh                       ← Service controller (start/stop/restart/status/view/logs)
│   ├── run.sh                        ← User-friendly wrapper for init.sh
│   ├── protect-my-ip.sh              ← Whitelist admin IP before blocking
│   ├── mutt.sh                       ← Send threat_ip.log via mutt
│   └── .env.example                  ← Configuration template
│
└── analysis_output/                  ← Runtime-generated (auto-created)
    ├── threat_ip.log                 ← Human-readable threat scorecard
    ├── threat_ip.json                ← JSON threat data
    ├── ips_detected.txt              ← Detected attacker IPs
    ├── firewall_rules.log            ← Firewall action audit trail
    ├── output.log                    ← Detection event log
    ├── main.log                      ← main.py stdout/stderr
    ├── firewall.log                  ← firewall_auto_ip_blocker.py stdout/stderr
    ├── main.pid                      ← PID of main.py process
    └── firewall.pid                  ← PID of firewall blocker process
```

---

## ⚡ Quick Start

```bash
git clone https://github.com/MonuJangra-git/MJ-IPguard.git
cd SSH_log_monitoring-alerting_and_banning_/working

cp .env.example .env
# Edit .env with your email credentials

pip install -r ../requirements.txt

bash run.sh
# Choose option 1 to start
```

---

## 🔧 Installation & Setup

**Requirements:**
- Linux with `/var/log/auth.log` (Debian/Ubuntu) or `/var/log/secure` (RHEL/CentOS)
- Python 3.x
- `firewalld` installed and accessible
- Root/sudo access
- `mutt` (optional, for email via mutt)

**Install dependencies:**
```bash
pip install -r requirements.txt
# Installs: regex, matplotlib, Flask
```

**Install firewalld (if missing):**
```bash
sudo apt install firewalld -y      # Debian/Ubuntu
sudo yum install firewalld -y      # RHEL/CentOS
```

---

## ⚙️ Configuration

Copy `.env.example` to `.env` inside `working/` and fill in your values:

```env
ALERT_EMAIL_SENDER=your_sender@example.com
ALERT_EMAIL_PASSWORD=your_app_password_here
ALERT_EMAIL_RECIPIENT=your_recipient@example.com
LOG_FILE_PATH=/var/log/auth.log
```

> Use a Gmail App Password, not your account password. Generate one at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords).

The `.env` file is excluded from git via `.gitignore`.

---

## 📊 Usage

From the `working/` directory, just run:

```bash
bash run.sh
```

An interactive menu appears — no need to remember any command names:

```
======================================================
        🛡️  MJ-IPguard — SSH Attack Defense
======================================================

  1)  Start          — Setup firewall & start all services
  2)  Stop           — Stop all running services
  3)  Restart        — Restart all services
  4)  Status         — Check if services are running
  5)  Web Dashboard  — Live threat chart at http://localhost:8080
  6)  Terminal Chart — Live threat chart in terminal window
  7)  Live Logs      — Stream main service logs (Ctrl+C to exit)
  8)  Firewall Logs  — Stream firewall action logs (Ctrl+C to exit)
  9)  Recent Threats — Show last 20 detected threats
  0)  Exit

Choose an option [0-9]:
```

Type a number and press Enter. That's it.

**What option `1` (Start) does step by step:**
1. Checks for root
2. Installs `firewalld` if missing
3. Creates `analysis_output/` if missing
4. Runs `protect-my-ip.sh` to whitelist your IP
5. Adds SSH rate-limit rule: `10/minute`
6. Reloads firewall
7. Runs `firewall-auto-setup.py` to verify firewalld is active
8. Starts `main.py` as a background process (PID saved to `main.pid`)
9. Starts `firewall_auto_ip_blocker.py` as a background process (PID saved to `firewall.pid`)

---

## 📁 Output Files

| File | Description |
|---|---|
| `threat_ip.log` | Scorecard with total attacks and last 10 logs per category |
| `threat_ip.json` | JSON with `total_attacks` and per-category `failed_attempts` + `recent_logs` |
| `ips_detected.txt` | One IP per line, consumed by the firewall blocker |
| `firewall_rules.log` | Timestamped log of every firewall add/skip/fail action |
| `output.log` | Appended log of every pattern match detected |
| `main.log` | stdout/stderr of main.py |
| `firewall.log` | stdout/stderr of firewall_auto_ip_blocker.py |

**Sample `threat_ip.json`:**
```json
{
    "total_attacks": 154,
    "attack_details": {
        "brute_force": { "failed_attempts": 50, "recent_logs": [...] },
        "lockout":     { "failed_attempts": 56, "recent_logs": [...] },
        "pam_failure": { "failed_attempts": 48, "recent_logs": [...] },
        "other_failed":{ "failed_attempts": 0,  "recent_logs": [] }
    }
}
```

---

## 🔍 Visualization

Both visualization modes are accessible from the interactive menu:

**Option 6 — Terminal chart** (requires a desktop environment):
```
bash run.sh  →  choose 6
```
Opens a matplotlib window with a color-coded bar chart that refreshes every 5 seconds.

**Option 5 — Web dashboard** (headless-friendly):
```
bash run.sh  →  choose 5
# Open http://localhost:8080 in your browser
```
Serves a PNG bar chart via Flask, auto-refreshing every 5 seconds. Accessible remotely if port 8080 is open.

---

## 🛡️ Security Features

- No credentials in source code — all loaded from `.env`
- `.env` excluded from git via `.gitignore`
- Admin IP whitelisted before any blocking rules are applied (`protect-my-ip.sh`)
- SSH rate-limiting applied at startup (`10/minute`)
- Strict IPv4 regex validation before any `firewall-cmd` call
- Set-based deduplication prevents redundant firewall rules
- `firewall-cmd` rules are runtime (not `--permanent`) by default — reboot clears them

---

## 📈 Market Relevance — Why This Matters

### The Problem at Scale

SSH brute-force attacks are not rare edge cases — they are constant background noise on any internet-facing Linux server. According to public threat intelligence reports:

- Exposed SSH ports receive automated login attempts **within minutes** of going online
- The majority of compromised servers are breached via **credential stuffing and brute-force**, not zero-days
- Most small teams and individual developers **cannot afford** Splunk, Datadog, or CrowdStrike — licenses start at hundreds to thousands of dollars per month

### Where MJ-IPguard Fits

The security tooling market has a clear gap between:

| Tier | Tools | Cost | Complexity |
|---|---|---|---|
| Enterprise SIEM | Splunk, IBM QRadar, Microsoft Sentinel | $$$$ | High — dedicated security team needed |
| Mid-market | Datadog Security, Elastic SIEM | $$$ | Medium — cloud setup required |
| Open-source heavy | Wazuh, OSSEC, Suricata | Free but complex | High — agents, rules, dashboards to configure |
| **MJ-IPguard** | **This project** | **Free** | **Low — one command to start** |

MJ-IPguard targets the gap that affects the most servers globally: **solo developers, small startups, VPS hosters, and sysadmins** who need real protection today without a week of setup.

### Why Automated Blocking Matters

A human cannot respond to SSH attacks fast enough. A typical brute-force bot attempts **hundreds of passwords per minute**. Manual IP banning after the fact is reactive and ineffective. MJ-IPguard's firewall blocker runs as a daemon and blocks IPs within **3 seconds** of detection — before the attacker can succeed.

### Why JSON Output Matters

Structured output (`threat_ip.json`) means MJ-IPguard is not a dead-end tool. The data it produces can be:
- Fed into a custom dashboard or Grafana
- Parsed by a cron job to generate weekly reports
- Consumed by a webhook to notify Slack or PagerDuty
- Used as input for threat intelligence correlation

This makes MJ-IPguard a **building block**, not just a standalone script.

### Real-World Applicability

- A VPS running a personal project gets scanned by bots constantly — MJ-IPguard handles this silently in the background
- A startup with 2–3 Linux servers needs security without a dedicated SecOps team — MJ-IPguard covers SSH attack surface with one `bash run.sh start`
- A developer learning security engineering gets a real, working detection + response pipeline to study and extend
- A sysadmin managing multiple servers can deploy MJ-IPguard on each and receive consolidated email alerts

---

## 🆚 MJ-IPguard vs Alternatives

| Feature | MJ-IPguard | fail2ban | Wazuh | Splunk SIEM |
|---|---|---|---|---|
| Real-time SSH detection | ✅ | ✅ | ✅ | ✅ |
| Auto IP blocking via firewalld | ✅ | ✅ | ✅ | ❌ (needs integration) |
| Email alerting | ✅ | ✅ | ✅ | ✅ |
| JSON structured output | ✅ | ❌ | ✅ | ✅ |
| Web visualization dashboard | ✅ (Flask) | ❌ | ✅ (complex) | ✅ (expensive) |
| Admin IP self-protection | ✅ (built-in) | ❌ | ❌ | ❌ |
| SSH rate limiting on setup | ✅ (auto) | ❌ | ❌ | ❌ |
| Zero config to start | ✅ (one command) | ❌ (jail config) | ❌ (agent setup) | ❌ (cloud setup) |
| Cost | Free | Free | Free | $$$$ |
| Python — easy to extend | ✅ | ❌ (config-based) | ❌ (complex) | ❌ |
| Runs on minimal VPS | ✅ | ✅ | ❌ (heavy) | ❌ (cloud) |

> fail2ban is the closest alternative. MJ-IPguard differentiates with built-in JSON analytics, a web dashboard, admin self-protection, and a fully Python codebase that is easy to read and extend.

---

## 🎯 Who Should Use This

- **VPS / cloud server owners** — any internet-facing Linux box running SSH
- **Solo developers and hobbyists** — protect your side projects without enterprise overhead
- **Small startups** — cover your SSH attack surface before you can afford a SecOps team
- **Students learning cybersecurity** — a real, working detection + response pipeline to study
- **Sysadmins** — lightweight daemon that runs quietly and emails you when something is wrong
- **CTF / homelab enthusiasts** — understand what real SSH attack logs look like

---

## 🔮 Possible Future Enhancements

These are natural extensions of the current architecture — the codebase is structured to support them:

- **Persistent firewall rules** — add `--permanent` flag to `firewall-cmd` calls so blocks survive reboots
- **Configurable alert threshold** — move the `500` attack threshold from hardcoded to `.env`
- **Slack / webhook notifications** — add a `notify_handler.py` alongside `email_handler.py`
- **IPv6 blocking support** — extend the IP regex and firewall rules to cover `::1` and other IPv6 addresses currently detected but not blocked
- **Whitelist file** — a `whitelist.txt` that `firewall_auto_ip_blocker.py` checks before blocking
- **Grafana integration** — expose `/metrics` endpoint from `visualize_web.py` in Prometheus format
- **Multi-server support** — aggregate `threat_ip.json` from multiple servers into a central dashboard
- **Log rotation handling** — detect when `auth.log` is rotated and reset the file position
- **Systemd service unit** — replace `nohup` with a proper `.service` file for auto-restart on crash

---

## 🐛 Troubleshooting

**Services not starting:**
```bash
bash run.sh status
cat analysis_output/main.log
cat analysis_output/firewall.log
```

**No IPs being blocked:**
- Check `analysis_output/ips_detected.txt` has entries
- Check `analysis_output/firewall_rules.log` for errors
- Ensure `firewalld` is running: `systemctl status firewalld`

**Email not sending:**
- Verify `.env` credentials are correct
- For Gmail, use an App Password (not your account password)
- Check `analysis_output/output.log` for SMTP errors

**Wrong log file path:**
- Default is `/var/log/auth.log` (Debian/Ubuntu)
- For RHEL/CentOS, change `file_name` in `main.py` to `/var/log/secure`

**`firewalld` not found:**
```bash
sudo apt install firewalld -y   # or yum
sudo systemctl enable --now firewalld
```
## 🤔 FAQ: Why Not Just Whitelist IPs in firewalld?

This is the most common question — and a valid one. If your server is accessed by only 2–3 people with **static IPs**, a default-deny whitelist **is** the stronger approach. Use it.

But most real-world servers don't live in that clean scenario. Here's why MJ-IPguard exists **alongside**, not instead of, IP whitelisting:

### 1. Dynamic IPs Break Whitelists

Most developers and remote workers connect from residential ISPs with **dynamic IPs** that change every few days. Every change means:

- A support ticket or Slack message
- A manual `firewall-cmd --remove-rich-rule` + `--add-rich-rule` update
- A risk of accidental lockout if the admin's IP also changed

MJ-IPguard eliminates that operational cost — legitimate users connect freely while attackers get blocked within **3 seconds** of detection.

### 2. You Don't Always Know Who Needs Access

New hires, contractors, CI/CD runners, third-party monitoring agents, emergency responders — these connections come from IPs you didn't know about yesterday. A strict whitelist blocks **all of them** until someone manually intervenes, potentially causing hours of downtime during an incident.

### 3. Whitelisting Gives You Zero Visibility

A default-deny rule **silently drops packets**. You never know someone was attacking you.

| What you get | Whitelist Only | MJ-IPguard |
|---|---|---|
| Attack count | ❌ | ✅ `threat_ip.json` |
| Attack categories | ❌ | ✅ brute_force / lockout / pam_failure |
| Timestamps | ❌ | ✅ Every event logged |
| Last 10 logs per category | ❌ | ✅ `threat_ip.log` |
| Email alerts | ❌ | ✅ Every 500 attacks |
| Audit trail for compliance | ❌ | ✅ `firewall_rules.log` |

A whitelist gives you **none of this**.

### 4. A Compromised Whitelisted IP Bypasses Everything

If a developer's home network is compromised and the attacker obtains their credentials, the attacker connects from a **whitelisted IP**. Firewalld lets them straight through.

MJ-IPguard would still detect the brute-force behavior — wrong passwords, PAM failures, lockout events — and block that IP based on **behavior, not identity**.

> **Whitelisting trusts the IP. MJ-IPguard watches the behavior.**

### 5. Compliance Requires Logs, Not Just Blocks

SOC 2, ISO 27001, PCI-DSS — most compliance frameworks ask:

> *"Can you show us the logs of what was attempted, when, and how you responded?"*

A silent firewall drop doesn't produce that evidence. MJ-IPguard's structured JSON and log output gives you a **complete, timestamped audit trail** ready for review.

### 6. Defense in Depth — Layers, Not Walls

Security is never a single mechanism. The strongest setups use multiple layers:

| Layer | Mechanism | Handles |
|---|---|---|
| **Layer 1** | firewalld whitelist | Known static IPs (office, CI/CD, monitoring) |
| **Layer 2** | MJ-IPguard | Dynamic users, unknown sources, compromised whitelisted IPs |
| **Layer 3** | SSH key-only auth | Eliminates password brute-force entirely |

> MJ-IPguard already supports this hybrid model — `protect-my-ip.sh` whitelists the admin's current IP **before** any blocking rules are applied. A planned enhancement will add a `whitelist.txt` that the firewall blocker checks before banning, so known IPs are never blocked even if they trigger detection patterns.

### TL;DR

| Scenario | Best Approach |
|---|---|
| 2–3 static IPs, no dynamic users | Whitelist only ✅ |
| Dynamic IPs, remote team, growing startup | MJ-IPguard ✅ |
| Compliance required, audit logs needed | MJ-IPguard ✅ |
| Maximum security | Whitelist **+** MJ-IPguard **+** key-only auth ✅✅✅ |

**Whitelist what you can. Let MJ-IPguard catch everything else.**
## Security Recommendation

MJ-IPguard is a defense-in-depth tool, not a replacement for SSH hardening.

Where operationally possible, protect SSH using one or more of:

- VPN-only or bastion-host access
- Source-IP allowlisting in firewalld
- SSH key authentication
- Disabling password authentication
- Disabling direct root login
- MFA for privileged access

MJ-IPguard is most useful when static source-IP allowlisting is not practical or when additional monitoring, alerting, and automated response are required.
---

## 📝 License

MIT License — free to use, modify, and distribute.

---

> **Author:** Monu Jangra  
> **GitHub:** [MonuJangra-git](https://github.com/MonuJangra-git)  
> **LinkedIn:** Monu Jangra  
> ⭐ Star this project if it helped you!
