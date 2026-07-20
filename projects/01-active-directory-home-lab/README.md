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
| **Internal Network** | 172.16.0.0/16 --- DHCP range 172.16.0.100-200 |
| **Date** | July 2026 |
| **Features Covered** | AD DS, OUs & Users, PowerShell Automation, NAT, DHCP, Domain Join |

Identity & Access Management \| Network Infrastructure

## What is this project, and why did I build it?

Almost every mid-to-large company in the world runs on Microsoft Active Directory. It is the system that decides who can log into which computer, what folders someone can access, how passwords are enforced, and how thousands of employees get connected to a network without an IT admin manually setting up every single machine by hand.

Rather than just reading about how this works, I built the entire system myself, from an empty virtual machine to a fully working small-scale corporate network. This document walks through exactly what I built, in the order I built it, and explains --- in plain language --- what each piece does and why it exists.

*If you have ever wondered what actually happens behind the scenes when you log into a work computer with a company email and password, this project is a working model of exactly that.*

## The network I built

The lab simulates a small company with two machines: one server that runs the entire network (the Domain Controller), and one employee laptop (the client) that connects to it. The diagram below shows how everything is connected.

*The overall network layout: the Domain Controller (DC) bridges the internet and an isolated internal network, while the client machine only ever talks to the DC.*

The Domain Controller has two network connections. One faces the internet (the same way your home router does), and the other faces a private, isolated internal network that only the lab\'s own machines belong to. Every internal machine --- including the client laptop --- gets its internet access, its network address, and its login credentials from this one server.

What I actually configured

-   Active Directory Domain Services (AD DS) --- the directory that stores every user and computer account, running as a brand-new domain: mydomain.com

-   DHCP Server --- automatically hands out network addresses to every device that joins the internal network

-   Routing and Remote Access (NAT) --- lets internal devices reach the internet through the server, the same way a home router shares one internet connection across every device in a house

-   Organizational Units and bulk user accounts --- a realistic structure of departments and employees, created partly by hand and partly through PowerShell automation

-   A domain-joined Windows 10 client --- a virtual employee laptop that authenticates against the server, proving the whole system actually works end-to-end

Step 1 --- Preparing the network connection

*Before any server software is installed, the machine needs a fixed, predictable address on the network --- the same way a business needs a fixed street address before it can receive mail.*

Every device on a network is identified by an IP address. Most home devices get an address automatically and it can change over time, which is fine for a laptop, but not for a server that other machines need to find reliably. So the very first task was to identify the network adapter that would serve the internal network, and give it a permanent, unchanging address.

1.1 Finding the right network adapter

*The server has two network adapters. The second one, still \'unidentified\' at this point, is the one that will become the internal network connection.*

> **Why this matters:** Servers commonly have more than one network connection so they can separate \'public-facing\' traffic from \'internal-only\' traffic --- exactly like a business might have a public reception desk and a separate staff-only back office.

1.2 Opening the adapter\'s settings

*The adapter is renamed to INTERNAL_NETWORK for clarity, and its IPv4 settings are opened for editing.*

1.3 Assigning a permanent address

*The adapter is given the fixed address 172.16.0.1, with the server also set to use itself (127.0.0.1) for DNS lookups.*

> **Why this matters:** This address (172.16.0.1) becomes the \'home base\' for the entire internal network --- every other device will be configured relative to this one fixed point. Pointing DNS at itself (127.0.0.1) is standard for a server that will also be resolving domain names for the network, which happens automatically once Active Directory is installed.

Step 2 --- Turning the server into a Domain Controller

*This is the single most important step in the whole project. Installing Active Directory Domain Services transforms an ordinary Windows Server into the authority that every other computer on the network will trust and take instructions from.*

In practical terms, this is the moment the server stops being just \'a computer\' and becomes the equivalent of a company\'s central HR and IT record system combined --- it will hold every username, every password policy, and every rule about who can access what.

2.1 Starting the role installation

*From Server Manager, the \'Add roles and features\' wizard is launched --- this is how new capabilities get installed on a Windows Server.*

2.2 Choosing Active Directory Domain Services

*Active Directory Domain Services (AD DS) is selected from the list of available server roles.*

> **Why this matters:** A Windows Server can run dozens of different roles --- file storage, web hosting, remote access, and more. AD DS is the specific role that adds directory and identity management capability.

2.3 Promoting the server

*After installation, Server Manager prompts to \'Promote this server to a domain controller\' --- this is the actual activation step.*

2.4 Creating a brand-new domain

*The configuration wizard is set to \'Add a new forest\', with the domain named mydomain.com.*

> **Why this matters:** Choosing \'new forest\' means this server is establishing an entirely new, independent company network from scratch --- as opposed to joining a domain that already exists elsewhere. mydomain.com now becomes the unique name that identifies this organization\'s network, similar to how a company\'s website domain identifies it on the internet.

Step 3 --- Organizing users with Active Directory Users and Computers

*Once the domain exists, it needs structure. Active Directory Users and Computers (ADUC) is the main console administrators use daily to manage who belongs to the network.*

3.1 Opening the management console

*ADUC is opened from the Windows Administrative Tools menu --- this becomes the day-to-day control panel for the domain.*

3.2 Creating an Organizational Unit

*Right-clicking the domain to create a New Organizational Unit (OU).*

*Selecting Organizational Unit from the object type menu.*

3.3 Naming the OU

*The new OU is named ADMINS, with accidental-deletion protection enabled.*

> **Why this matters:** An Organizational Unit is a folder-like container used to group similar accounts together --- for example, all IT administrators in one OU, all sales staff in another. This lets a real company apply different rules, permissions, and policies to different groups of employees, instead of treating every account identically.

3.4 Creating an individual user account

*Creating a new User object inside an OU through the right-click menu.*

This is the same action an IT department performs every time a new employee is hired --- creating them a username, a temporary password, and placing their account in the correct department folder.

Step 4 --- Automating account creation with PowerShell

*Real companies don\'t create employee accounts one at a time by clicking through menus --- they automate it. To simulate that realistically, I used a PowerShell script to generate multiple accounts at once.*

4.1 The reference script

*A publicly available PowerShell script that generates random employee names and creates a corresponding AD account for each one.*

4.2 Running the automated account creation

*The script executing in PowerShell ISE --- reading a list of names, creating a dedicated OU, and provisioning a user account for every name on the list.*

> **Why this matters:** This mirrors exactly how real onboarding works at scale: HR provides a list of new hires, and a script (or an identity management system) provisions their accounts automatically, rather than an admin manually typing in hundreds of names. It\'s a small demonstration of IT automation, a skill that saves companies enormous amounts of time.

Step 5 --- Giving the internal network internet access

*The internal network is deliberately isolated --- none of its devices can talk to the internet directly. To fix that without exposing every device individually, the server was configured to share its own internet connection, the same way a household router does.*

5.1 Installing the Remote Access role

*The Remote Access server role is added through Server Manager.*

5.2 Selecting the routing service

*DirectAccess/VPN (RAS) and Routing services are selected as part of the Remote Access role.*

5.3 Opening Routing and Remote Access

*The Routing and Remote Access console is opened from the server\'s Tools menu.*

5.4 Enabling the service

*Right-clicking the server to Configure and Enable Routing and Remote Access.*

5.5 Choosing Network Address Translation (NAT)

*NAT is selected as the configuration type --- this is the same technology used by almost every home Wi-Fi router.*

> **Why this matters:** Network Address Translation (NAT) allows many devices behind one connection to all share a single public internet address, while keeping their internal addresses private. It\'s exactly how your phone, laptop, and smart TV all get internet through one home router. Here, it lets the internal lab network reach the internet through the server.

5.6 Assigning the public and private interfaces

*The internet-facing adapter (INTER_NET) is set as the \'public\' side, while the internal adapter remains the \'private\' side that NAT protects.*

Step 6 --- Automatic addressing with DHCP

*Manually typing in a unique network address on every single device would not scale to a real office of hundreds of employees. DHCP solves this by handing out addresses automatically.*

6.1 Installing the DHCP role

*The DHCP Server role is added via Server Manager.*

6.2 Opening the DHCP console

*The DHCP management console is opened from the Tools menu.*

6.3 Creating a new scope

*A New Scope is started --- a \'scope\' defines the range of addresses DHCP is allowed to give out.*

6.4 Naming the scope

*The scope is named 172.16.0.100-200, matching the range it will manage.*

6.5 Defining the address range

*The scope is configured to hand out addresses between 172.16.0.100 and 172.16.0.200.*

> **Why this matters:** This means the first 99 addresses on the internal network (172.16.0.1--99) are reserved for infrastructure like the server itself, while everyday devices --- employee laptops, printers, etc. --- automatically receive an address from the 100--200 range the moment they connect.

6.6 Setting the default gateway

*The default gateway --- the address every device should send internet-bound traffic to --- is configured as part of the scope.*

**A lesson learned:** during this step, the gateway was mistakenly entered as 172.168.0.1 instead of the correct 172.16.0.1 --- a simple one-digit typo. It\'s a good example of how a tiny data-entry mistake in network configuration can silently break internet access for every device on the network, and why configuration values should always be double-checked against the actual server settings. This was identified during review and corrected.

Step 7 --- Connecting an employee laptop to the network

*With the server fully configured, the final test was to join an actual client machine to the domain --- this is the moment that proves the entire system works together.*

7.1 Preparing to rename the client

*Opening the \'Rename this PC (advanced)\' option in Windows Settings on the client machine.*

7.2 Opening computer identification settings

*The System Properties window, where a computer\'s name and domain membership are managed.*

7.3 Joining the domain

*The computer is renamed Client1 and configured to join the domain mydomain.com instead of a standalone Workgroup.*

> **Why this matters:** Joining a domain is the process that hands control of a computer over to central IT management. Once joined, the company (via the Domain Controller) can enforce password policies, push software, and manage settings --- the computer is no longer an independent island.

7.4 Confirming the computer registered correctly

*Back on the server, CLIENT1 now appears automatically in the Computers container inside Active Directory --- proof the domain join was successful.*

7.5 Logging in as a real employee account

*Logging into Client1 using a domain user account (aabrev), authenticating against MYDOMAIN.*

This final login is the payoff of the entire project: a username and password, created earlier through the PowerShell automation script, is now being used to log into a completely separate machine --- proving that identity, addressing, DNS, and network routing are all working together correctly, exactly as they would in a real office network.

What this project demonstrates

Taken together, this lab is a working, end-to-end model of the infrastructure that sits behind almost every corporate login screen in the world. Specifically, it demonstrates practical, hands-on ability to:

-   Design and configure a segmented network with separate public and private-facing connections

-   Deploy and administer Active Directory Domain Services, including domain and forest creation

-   Structure an organization\'s user base using Organizational Units and role-based grouping

-   Automate repetitive administrative tasks using PowerShell scripting

-   Configure NAT-based internet sharing and DHCP addressing --- the same technologies used in home routers and enterprise networks alike

-   Domain-join and authenticate a client machine, validating the environment end-to-end

-   Identify and correct a real misconfiguration (the gateway typo), reflecting the kind of troubleshooting IT professionals do daily

This project was built entirely in a virtualized lab environment for learning purposes, and forms the foundation for a follow-up project --- layering security monitoring tools (Sysmon and Splunk) on top of this same network to detect and investigate suspicious activity, which is documented separately.
