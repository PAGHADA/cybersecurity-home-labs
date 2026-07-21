# Wazuh — Open Source SIEM/XDR Deployment

**Complete Project Walkthrough & Notes**

| Field | Detail |
|---|---|
| **Reporter** | Yash Paghada |
| **Platform** | Wazuh v4.12 (Server) + v4.14.5 (Agent) |
| **Server** | Ubuntu 26.04 LTS — IP: 192.168.1.24 |
| **Agent** | Windows 11 Pro — WindowsAgent (ID: 001) |
| **Date** | June 15, 2026 |
| **Features Covered** | Installation, FIM, Auth Monitoring, Vulnerability Detection |

## What is Wazuh?

Wazuh is a free, open-source security platform that provides unified protection across endpoints and cloud workloads. It combines Security Information and Event Management (SIEM) with Extended Detection and Response (XDR) capabilities.

| Feature | Description |
|---|---|
| **SIEM** | Collects and analyzes security logs from all connected machines in real time |
| **XDR** | Detects threats across endpoints, networks and cloud |
| **FIM** | File Integrity Monitoring — detects any file changes (added, modified, deleted) |
| **Vulnerability Detection** | Scans agents for known CVEs and software vulnerabilities |
| **Authentication Monitoring** | Tracks login successes, failures and brute force attempts |
| **Agent-Server Model** | Central Wazuh server collects data from agents installed on monitored machines |

| Component | Role |
|---|---|
| **Wazuh Server** | Central brain — receives, analyzes and stores all security data |
| **Wazuh Indexer** | Stores and indexes all security events (based on OpenSearch) |
| **Wazuh Dashboard** | Web UI at https://192.168.1.24 — visualizes all alerts and data |
| **Wazuh Agent** | Installed on monitored machines — sends logs/events to the server |

## Phase 1: Wazuh Server Installation

The Wazuh server is installed on an Ubuntu 26.04 LTS machine. Connected via SSH from Kali Linux and ran the official Wazuh installation script, which automatically installs all required components.

**1.1 — Connect via SSH**
```
ssh kali@192.168.1.24
```
SSH provides an encrypted connection to the Ubuntu server so all commands are sent securely.

**1.2 — Add Wazuh GPG key**
```
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg
```
A GPG key verifies that the Wazuh packages downloaded are authentic and haven't been tampered with — a security best practice before installing any third-party software.

**1.3 — Run the installation script**
```
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh && sudo bash ./wazuh-install.sh -a -i
```
The `-a` flag installs ALL components (Server, Indexer, Dashboard); `-i` ignores hardware requirement warnings. The script handles all dependencies, configuration, and service startup automatically. The web interface runs on port 443 (HTTPS).

**1.4 — Access the dashboard** at `https://192.168.1.24`. Default credentials are provided at the end of the installation script output.

## Phase 2: Windows Agent Setup

A Wazuh agent is lightweight software installed on machines to be monitored. It collects security data (logs, file changes, network activity) and sends it to the server in real time.

- **2.1** Downloaded the Windows agent installer from the official Wazuh documentation.
- **2.2** On the server, ran `sudo /var/ossec/bin/manage_agents` — the Wazuh CLI tool for managing agents — and chose option A to add a new agent.
- **2.3** Named the agent `WindowsAgent` and provided the Windows machine's IP.
- **2.4** Confirmed registration (Agent ID `001`) and extracted the authentication key with option E — a long Base64-encoded string that authenticates the agent to the server.
- **2.5** On the Windows machine, opened the Wazuh Agent GUI, entered the Manager IP (`192.168.1.24`), pasted the authentication key, and restarted the agent service — status changed from **Not Running** to **Connected and Running**.

## Phase 3: File Integrity Monitoring (FIM)

FIM is one of Wazuh's most powerful features: it monitors specified directories for ANY changes — files added, modified, or deleted — and sends real-time alerts to the dashboard, critical for detecting unauthorized changes to important files.

- **3.1** Edited `ossec.conf` (the Wazuh agent's main config file on Windows) to add a monitored directory with `realtime="yes"`:
```xml
<directories realtime="yes">C:\Users\INET COMPUTER\Test</directories>
```
- **3.2** Created a test folder and added a file (`yash.txt`) inside it to trigger FIM alerts.
- **3.3** The dashboard immediately showed FIM events — file added, renamed, deleted — each tagged with file path, action type, and alert level:

| Event | Detail |
|---|---|
| **File Added** | `yash.txt` — Level 5 |
| **File Added** | `new text document.txt` — Level 5 |
| **File Deleted** | `new text document.txt` — Level 7 |
| **Detection** | Real-time — alerts appeared within seconds |

## Phase 4: Authentication Monitoring

Wazuh automatically monitors all authentication events on connected agents — successful logins, failed attempts, account lockouts, and privilege escalations — critical for detecting brute force attacks and unauthorized access.

A failed login attempt was simulated (Win+L with a wrong password), and the dashboard reflected it in the 24-hour authentication summary for the Windows agent:

| Metric | Value |
|---|---|
| **Total Events** | 553 security events in 24 hours |
| **Level 12+ Alerts** | 0 critical alerts |
| **Authentication Failures** | 3 failed login attempts detected |
| **Authentication Success** | 15 successful logins recorded |
| **Alert Groups** | windows, WEF, windows_security, authentication_success, syscheck |

| Event ID | Meaning |
|---|---|
| **4625** | Failed logon attempt — triggers authentication failure alert |
| **4624** | Successful logon |
| **4740** | Account locked out — high severity alert |
| **4648** | Logon using explicit credentials |

## Phase 5: Vulnerability Detection

Wazuh's Vulnerability Detection module scans all software installed on connected agents and compares it against the National Vulnerability Database (NVD) and other CVE databases, rating results by severity (CVSS score).

Wazuh automatically scanned the Windows 11 agent and found:

| Severity | Count |
|---|---|
| **Critical** | 1 — immediate action required |
| **High** | 6 — should be patched soon |
| **Medium** | 6 — monitor and plan remediation |
| **Low** | 0 |
| **Total CVEs Found** | 13 vulnerabilities on the Windows 11 agent |

Notable CVEs (CVE-2019-12874, CVE-2019-13602, CVE-2019-13962, CVE-2019-19721, CVE-2019-5439) were found in two vulnerable packages: **VLC Media Player** and **MySQL Server 8.0**, on `Microsoft Windows 11 Pro 10.0.26200.8655`.

## Project Summary

This project demonstrated a complete Wazuh SIEM deployment from installation to active monitoring, successfully detecting and alerting on multiple security scenarios in real time.

| Phase | What Was Done |
|---|---|
| **1 — Installation** | Installed Wazuh server on Ubuntu 26.04 via official script — dashboard at https://192.168.1.24 |
| **2 — Agent Setup** | Registered Windows 11 agent (ID: 001) via `manage_agents`, imported auth key via GUI |
| **3 — FIM** | Configured real-time file monitoring — detected file add/delete instantly |
| **4 — Auth Monitoring** | Simulated failed logins — Wazuh detected 3 failures and 15 successes in 24h |
| **5 — Vuln Detection** | Automated scan found 13 CVEs — 1 Critical, 6 High in VLC and MySQL |

**Tools & commands used:** `ssh`, `curl` + `gpg --dearmor`, `wazuh-install.sh -a -i`, `manage_agents`, `ossec.conf`, `realtime="yes"`, the Wazuh Dashboard.

**Key concepts:** centralized security monitoring (SIEM), the agent-server model, File Integrity Monitoring, authentication/brute-force monitoring, and automated CVE-based vulnerability detection.

## Restart Procedure

Since this is a home lab, services need to be restarted after every reboot:

1. **On Ubuntu:** `sudo systemctl start wazuh-manager wazuh-indexer wazuh-dashboard`, then verify with `sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard` (expect `active (running)`).
2. **On Windows (as Administrator):** `sc start WazuhSvc`, then verify with `sc query WazuhSvc` (expect `STATE: 4 RUNNING`).
3. **On Ubuntu:** `sudo /var/ossec/bin/agent_control -l` to confirm both agents show Active (run this *after* starting the Windows agent).
4. **Dashboard:** Open `https://192.168.1.24`, log in as `admin`, and check Agents Management → Summary for `Windows11 → Active (green)`.

## Project Features Demonstrated

- Centralized Log Management — all security events from the Windows agent collected in one place
- Vulnerability Detection — automated CVE scanning found 1 Critical, 6 High vulnerabilities
- File Integrity Monitoring (FIM) — real-time detection of file add/modify/delete
- Security Event Monitoring — 553 security events captured and analyzed
- Failed Login Detection — 3 authentication failures detected from a wrong-password test
- Real-Time Alerting — instant alerts on the dashboard for all security events
- Endpoint Monitoring — complete visibility into Windows 11 machine activity
