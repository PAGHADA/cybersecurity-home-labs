# Cybersecurity Home Labs

Hands-on security engineering projects covering identity infrastructure, SIEM/XDR monitoring, threat detection, AI-assisted pentesting, and perimeter defense — built end-to-end in self-hosted and cloud lab environments.

**Author:** Yash Paghada
**Focus areas:** Security Operations (SOC) · SIEM/XDR · Identity & Access Management · Penetration Testing · Detection Engineering · AI-Assisted Security Automation

---

## About this repository

Each folder below is a self-contained project with its own detailed write-up: what was built, why, the exact steps and commands used, and what it demonstrates. These aren't tutorials followed passively — every lab was deployed, broken, and fixed hands-on, from empty virtual machines to fully working systems.

## Projects

| # | Project | What it demonstrates | Stack |
|---|---|---|---|
| 1 | [Active Directory Home Lab](./projects/01-active-directory-home-lab) | Enterprise identity & network infrastructure — AD DS, OUs/Users, PowerShell automation, DHCP/NAT, domain join | Windows Server 2019, Windows 10, VMware/VirtualBox |
| 2 | [Sysmon + Splunk Monitoring](./projects/02-sysmon-splunk-monitoring) | Endpoint telemetry pipeline into a SIEM — detailed activity logging and centralized log search | Sysmon, Splunk Enterprise |
| 3 | [Azure Honeypot & Microsoft Sentinel SOC Lab](./projects/03-azure-honeypot-sentinel-soc) | Live internet-facing honeypot capturing real-world attacks, centralized into a cloud SIEM with geo-enrichment | Azure VM, Microsoft Sentinel |
| 4 | [Penligent AI Pentesting Lab](./projects/04-penligent-ai-pentesting-lab) | AI-assisted penetration testing workflow against a vulnerable web app — recon, exploitation, reporting | DVWA, Penligent AI, Kali Linux |
| 5 | [Wazuh SIEM/XDR](./projects/05-wazuh-siem-xdr) | Open-source SIEM/XDR deployment — file integrity monitoring, auth monitoring, vulnerability detection | Wazuh (server + agent) |
| 6 | [AI-Powered SOC Triage Agent](./projects/06-ai-soc-triage-agent) | End-to-end detection-to-triage pipeline — packet capture, anomaly scoring, LLM-based alert enrichment with MITRE ATT&CK mapping | Python, tshark, Airia (GPT-5 Nano) |
| 7 | [SafeLine WAF Lab](./projects/07-safeline-waf-lab) | Reverse-proxy web application firewall in front of a vulnerable app — SSL, onboarding, live SQLi testing, rate-limiting | SafeLine WAF, DVWA |

## Skills demonstrated across these labs

- **Identity & Access Management:** Active Directory, Group Policy concepts, PowerShell automation
- **SIEM / SOC tooling:** Splunk, Microsoft Sentinel, Wazuh
- **Detection Engineering:** Sysmon telemetry, File Integrity Monitoring, authentication/brute-force monitoring, MITRE ATT&CK mapping
- **Offensive Security:** Web app penetration testing (DVWA), SQL injection, AI-assisted recon and exploitation
- **Defensive Security:** Web Application Firewall deployment and tuning, honeypot design and attacker telemetry analysis
- **Cloud:** Azure VM deployment, Network Security Groups, cloud-native SIEM
- **Automation & AI:** Python/tshark network monitoring, LLM-based SOC alert triage pipelines

---

*All labs were built and documented in self-hosted virtualization environments (VMware/VirtualBox) and Microsoft Azure.*
