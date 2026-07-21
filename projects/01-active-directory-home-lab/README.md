# Active Directory Home Lab — Enterprise Identity & Network Infrastructure

**Active Directory**

Enterprise Identity & Network Infrastructure

**Complete Project Walkthrough & Notes**

| Field | Detail |
|---|---|
| **Reporter** | Yash Paghada |
| **Platform** | Windows Server 2019 (DC) + Windows 10 (Client) |
| **Environment** | VMware Workstation / VirtualBox |
| **Domain** | mydomain.com |
| **Internal Network** | 172.16.0.0/16 — DHCP range 172.16.0.100-200 |
| **Date** | July 2026 |
| **Features Covered** | AD DS, OUs & Users, PowerShell Automation, NAT, DHCP, Domain Join |

Identity & Access Management | Network Infrastructure

## What is this project, and why did I build it?

Almost every mid-to-large company in the world runs on Microsoft Active Directory. It is the system that decides who can log into which computer, what folders someone can access, how passwords are enforced, and how thousands of employees get connected to a network without an IT admin manually setting up every single machine by hand.

Rather than just reading about how this works, I built the entire system myself, from an empty virtual machine to a fully working small-scale corporate network. This document walks through exactly what I built, in the order I built it, and explains — in plain language — what each piece does and why it exists.

*If you have ever wondered what actually happens behind the scenes when you log into a work computer with a company email and password, this project is a working model of exactly that.*

## The network I built

The lab simulates a small company with two machines: one server that runs the entire network (the Domain Controller), and one employee laptop (the client) that connects to it.

The Domain Controller has two network connections. One faces the internet (the same way your home router does), and the other faces a private, isolated internal network that only the lab's own machines belong to. Every internal machine — including the client laptop — gets its internet access, its network address, and its login credentials from this one server.

### What I actually configured

- **Active Directory Domain Services (AD DS)** — the directory that stores every user and computer account, running as a brand-new domain: mydomain.com
- **DHCP Server** — automatically hands out network addresses to every device that joins the internal network
- **Routing and Remote Access (NAT)** — lets internal devices reach the internet through the server, the same way a home router shares one internet connection across every device in a house
- **Organizational Units and bulk user accounts** — a realistic structure of departments and employees, created partly by hand and partly through PowerShell automation
- **A domain-joined Windows 10 client** — a virtual employee laptop that authenticates against the server, proving the whole system actually works end-to-end

## Step 1 — Preparing the network connection

Every device on a network is identified by an IP address. Most home devices get an address automatically and it can change over time, which is fine for a laptop, but not for a server that other machines need to find reliably. So the very first task was to identify the network adapter that would serve the internal network, and give it a permanent, unchanging address.

- **1.1** Identified the network adapter that would become the internal network connection.
- **1.2** Renamed it `INTERNAL_NETWORK` and opened its IPv4 settings.
- **1.3** Assigned it the fixed address `172.16.0.1`, with DNS pointed at itself (`127.0.0.1`).

> **Why this matters:** This address (172.16.0.1) becomes the 'home base' for the entire internal network — every other device will be configured relative to this one fixed point.

## Step 2 — Turning the server into a Domain Controller

This is the single most important step in the whole project. Installing Active Directory Domain Services transforms an ordinary Windows Server into the authority that every other computer on the network will trust and take instructions from.

- **2.1–2.2** Installed the AD DS role via Server Manager's 'Add roles and features' wizard.
- **2.3** Promoted the server to a domain controller.
- **2.4** Chose 'Add a new forest' and named the domain `mydomain.com`.

> **Why this matters:** Choosing 'new forest' means this server is establishing an entirely new, independent company network from scratch. `mydomain.com` becomes the unique name that identifies this organization's network.

## Step 3 — Organizing users with Active Directory Users and Computers

Once the domain exists, it needs structure. Active Directory Users and Computers (ADUC) is the main console administrators use daily to manage who belongs to the network.

- **3.1** Opened ADUC from the Windows Administrative Tools menu.
- **3.2–3.3** Created a new Organizational Unit named `ADMINS`, with accidental-deletion protection enabled.
- **3.4** Created an individual user account inside the OU.

> **Why this matters:** An Organizational Unit is a folder-like container used to group similar accounts together — e.g. all IT administrators in one OU, all sales staff in another — letting a real company apply different rules and permissions to different groups of employees.

## Step 4 — Automating account creation with PowerShell

Real companies don't create employee accounts one at a time by clicking through menus — they automate it. To simulate that realistically, I used a PowerShell script to generate multiple accounts at once, reading a list of names and provisioning a dedicated OU plus a user account for each one.

> **Why this matters:** This mirrors exactly how real onboarding works at scale: HR provides a list of new hires, and a script (or an identity management system) provisions their accounts automatically — a small demonstration of the IT automation skill that saves companies enormous amounts of time.

## Step 5 — Giving the internal network internet access

The internal network is deliberately isolated — none of its devices can talk to the internet directly. To fix that without exposing every device individually, the server was configured to share its own internet connection, the same way a household router does.

- **5.1–5.2** Added the Remote Access server role with the Routing service.
- **5.3–5.4** Enabled Routing and Remote Access from the Tools menu.
- **5.5** Selected Network Address Translation (NAT) as the configuration type.
- **5.6** Set the internet-facing adapter as 'public' and the internal adapter as the 'private' side that NAT protects.

> **Why this matters:** NAT allows many devices behind one connection to all share a single public internet address while keeping their internal addresses private — the same technology used by almost every home Wi-Fi router.

## Step 6 — Automatic addressing with DHCP

Manually typing in a unique network address on every single device would not scale to a real office of hundreds of employees. DHCP solves this by handing out addresses automatically.

- **6.1–6.2** Installed the DHCP Server role and opened the DHCP console.
- **6.3–6.5** Created a new scope named `172.16.0.100-200`, configured to hand out addresses in that range.
- **6.6** Set the default gateway for the scope.

> **Why this matters:** The first 99 addresses on the internal network (172.16.0.1–99) are reserved for infrastructure like the server itself, while everyday devices — employee laptops, printers, etc. — automatically receive an address from the 100–200 range the moment they connect.

**A lesson learned:** during this step, the gateway was mistakenly entered as `172.168.0.1` instead of the correct `172.16.0.1` — a simple one-digit typo. It's a good example of how a tiny data-entry mistake in network configuration can silently break internet access for every device on the network, and why configuration values should always be double-checked. This was identified during review and corrected.

## Step 7 — Connecting an employee laptop to the network

With the server fully configured, the final test was to join an actual client machine to the domain — proving the entire system works together.

- **7.1–7.3** Renamed the client `Client1` and joined it to the domain `mydomain.com` instead of a standalone Workgroup.
- **7.4** Confirmed `CLIENT1` automatically appeared in the Computers container in Active Directory.
- **7.5** Logged into Client1 using a domain user account (`aabrev`), authenticating against `MYDOMAIN`.

This final login is the payoff of the entire project: a username and password, created earlier through the PowerShell automation script, is now being used to log into a completely separate machine — proving that identity, addressing, DNS, and network routing are all working together correctly, exactly as they would in a real office network.

## What this project demonstrates

Taken together, this lab is a working, end-to-end model of the infrastructure that sits behind almost every corporate login screen in the world. Specifically, it demonstrates practical, hands-on ability to:

- Design and configure a segmented network with separate public and private-facing connections
- Deploy and administer Active Directory Domain Services, including domain and forest creation
- Structure an organization's user base using Organizational Units and role-based grouping
- Automate repetitive administrative tasks using PowerShell scripting
- Configure NAT-based internet sharing and DHCP addressing — the same technologies used in home routers and enterprise networks alike
- Domain-join and authenticate a client machine, validating the environment end-to-end
- Identify and correct a real misconfiguration (the gateway typo), reflecting the kind of troubleshooting IT professionals do daily

This project was built entirely in a virtualized lab environment for learning purposes, and forms the foundation for a follow-up project — layering security monitoring tools (Sysmon and Splunk) on top of this same network to detect and investigate suspicious activity.
