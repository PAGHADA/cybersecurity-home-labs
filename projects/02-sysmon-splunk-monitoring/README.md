# Sysmon + Splunk — Endpoint Telemetry & SIEM Integration

**Complete Project Walkthrough & Notes**

| Field | Detail |
|---|---|
| **Reporter** | Yash Paghada |
| **Platform** | Sysmon v15.21 + Splunk Enterprise 10.4.1 |
| **Monitored Host** | DC (Windows Server 2019, mydomain.com) |
| **Sysmon Config** | SwiftOnSecurity sysmonconfig-export.xml |
| **Log Sources** | Security, System, Application (Sysmon in progress) |
| **Date** | July 2026 |
| **Features Covered** | Sysmon deployment, Splunk install, log ingestion, search |

Security Monitoring | SIEM | Endpoint Telemetry

## What is this project, and why did I build it?

This project is a direct continuation of the Active Directory home lab I built previously. Once a network is up and running, a natural next question for any IT or security professional is: how do you know what's actually happening on it? If an account is compromised, if malware runs on a machine, or if someone tries something they shouldn't — how would anyone find out?

The answer, in almost every real company, is a combination of two things: an agent that watches activity on each machine in detail, and a central system that collects and lets you search through that activity. This project builds exactly that pairing — Sysmon as the detailed activity sensor, and Splunk as the central search and analysis platform — on top of the Domain Controller from my earlier Active Directory build.

*If Active Directory is the system that decides who is allowed to do what, this project is the system that watches what everyone actually did.*

### The two tools involved

- **Sysmon (System Monitor)** — a free Microsoft Sysinternals tool that installs as a background service and records extremely detailed activity on a machine: every process that starts, every network connection made, every DNS lookup, and changes to sensitive parts of the system.
- **Splunk** — a Security Information and Event Management (SIEM) platform that collects logs from across a network into one place, so they can be searched, filtered, and analyzed instead of being checked machine-by-machine.

Together, these two tools form the foundation of a security monitoring pipeline: activity happens on a machine, Sysmon records it in detail, and Splunk makes that record searchable.

## Step 1 — Installing Sysmon, the activity sensor

Windows already keeps some logs by default, but they're often too generic to be useful for real investigation. Sysmon fills that gap by logging activity in far greater detail than Windows does on its own.

- **1.1** Downloaded Sysmon v15.21, published by Microsoft's Sysinternals team.
- **1.2** Chose the SwiftOnSecurity `sysmon-config` project on GitHub — a widely used, community-maintained configuration defining exactly what Sysmon should watch for.
- **1.3–1.4** Opened PowerShell as Administrator and ran `Sysmon64.exe -accepteula -i` with the config file to install it as a permanent background service.
- **1.5–1.6** Confirmed the `Sysmon64` service was running, and located its dedicated Operational log under Event Viewer → Applications and Services Logs → Microsoft → Windows.

> **Why this matters:** Sysmon on its own doesn't know what to look for — it needs a configuration file telling it which events matter. Rather than writing rules from scratch, I used SwiftOnSecurity's configuration, a well-known, actively maintained ruleset trusted by security practitioners worldwide, focused on capturing high-value security events without generating excessive noise. The `-i` flag installs Sysmon so it starts automatically on every boot, not just for the current session.

From this point on, every process launched, every network connection made, and every DNS lookup performed on this machine is being quietly recorded in the background — the raw material a SIEM platform like Splunk will later collect and make searchable.

## Step 2 — Installing Splunk, the central SIEM

Sysmon alone only stores logs locally on one machine, which doesn't scale — checking dozens of machines individually isn't practical. Splunk solves this by collecting logs centrally into one searchable platform.

- **2.1** Downloaded Splunk Enterprise 10.4.1 for Windows (free trial: 500 MB/day indexing, converts to a permanent free license after 60 days).
- **2.2–2.3** Launched Splunk Enterprise and logged into the web interface as Administrator at `http://localhost:8000`.

> **Why this matters:** Splunk is one of the most widely used SIEM platforms in the security industry, used by SOCs around the world to detect and investigate incidents. Learning it hands-on is directly transferable to real analyst work.

## Step 3 — Installing Splunk add-ons for Windows and Sysmon

Splunk can technically ingest any text-based log, but without the right add-on it won't understand the structure of Windows or Sysmon events — it would just see a wall of raw text instead of neatly organized fields.

- **3.1** Installed the Splunk Add-on for Sysmon via Splunk's 'Browse More Apps' marketplace.
- **3.2** Installed the Splunk Add-on for Microsoft Windows, which parses standard Windows Event Logs (Security, System, Application).

> **Why this matters:** These add-ons teach Splunk how to parse Sysmon's and Windows' specific event structures — process IDs, command lines, network destinations — into clean, individually searchable fields.

## Step 4 — Connecting Windows logs into Splunk

Installing the add-ons only teaches Splunk how to read the logs — it still needs to be told which logs on this machine to actually collect.

- **4.1** Opened Data Inputs from Splunk's Settings menu.
- **4.2** Selected 'Local event log collection' to pull Windows Event Logs directly from the same machine Splunk is running on.
- **4.3** Selected Security, System, and Application from the list of available Windows log channels.

> **Why this matters:** Security logs capture authentication activity (logins, logouts, permission changes) — exactly the kind of activity a security analyst cares about most, since it's often the first sign of a compromised account.

## Step 5 — Verifying data is flowing into Splunk

Configuration only matters if it actually works — the final step is to confirm real log data is landing inside Splunk and can be searched.

- **5.1** Opened the Search & Reporting app.
- **5.2** Ran the query `index=*` to view every indexed event — 590 events matched, confirming Windows log data (including DNS client and Service Control Manager activity from the domain controller) was flowing into Splunk successfully.

At this stage, standard Windows Security, System, and Application logs are confirmed flowing into Splunk and fully searchable. The Sysmon-specific log channel (`Microsoft-Windows-Sysmon/Operational`) required an additional manual step to register with Splunk, since it doesn't appear in Splunk's default local event log list — this is being finalized separately.

> **Note:** This is a realistic and common hurdle: newer, non-standard Windows event log channels aren't always exposed through a SIEM's default configuration UI, and often need to be registered manually through a configuration file (`inputs.conf`). Encountering and working through this is itself a normal part of real-world SIEM administration.

## What this project demonstrates

This project builds directly on top of the Active Directory lab and introduces the monitoring layer that any real security team would add to a live network. Specifically, it demonstrates hands-on ability to:

- Deploy Sysmon using an industry-standard, community-vetted configuration (SwiftOnSecurity)
- Understand and articulate the purpose of endpoint telemetry versus default OS logging
- Install and configure Splunk Enterprise, a leading commercial SIEM platform
- Install and apply Technology Add-ons to correctly parse vendor-specific log formats
- Configure local data inputs to bring Windows Event Logs into a central search platform
- Validate a monitoring pipeline end-to-end by confirming indexed, searchable data
- Troubleshoot a real configuration gap (the missing Sysmon channel) — the kind of practical problem-solving expected of a SOC analyst

Combined with the earlier Active Directory build, these two projects together form a small but complete model of enterprise IT: an identity and network infrastructure layer, and a security visibility layer sitting on top of it.
