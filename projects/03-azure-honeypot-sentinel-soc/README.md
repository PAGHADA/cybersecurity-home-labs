# Azure Honeypot & Microsoft Sentinel — SOC Home Lab

**Complete Project Walkthrough & Notes**

| Field | Detail |
|---|---|
| **Reporter** | Yash Paghada |
| **Platform** | Azure VM (Windows 11 Pro) + Microsoft Sentinel |
| **Date** | July 2026 |
| **Features Covered** | Azure Cloud, Microsoft Sentinel (SIEM) |

## 1. Project Overview

This project involved designing and deploying a live honeypot inside Microsoft Azure in order to attract, capture, and analyze real-world cyberattacks originating from the public internet. A deliberately exposed Windows virtual machine was placed online with no meaningful protection, and every interaction with it — successful or failed — was centralized into a Security Information and Event Management (SIEM) platform, Microsoft Sentinel, where it could be queried, enriched with contextual data, and visualized geographically.

The purpose of building a project like this is to simulate, in miniature, the kind of telemetry pipeline a real Security Operations Center (SOC) relies on every day: a data source generates raw events, those events are shipped to a central log store, an analyst or automated rule queries and filters that data, and the results are enriched and visualized so that decisions can be made quickly.

### 1.1 What is a Honeypot?

A honeypot is a decoy computer system that is intentionally made to look vulnerable or valuable so that attackers will target it instead of — or in addition to — real production systems. Because a honeypot has no legitimate business purpose, any traffic or activity it receives can be treated as suspicious by definition, which makes it an excellent, low-noise source of real attacker behavior for research and detection engineering.

In this project, the honeypot was a Windows 11 Pro virtual machine hosted in Azure. It was made deliberately vulnerable in two ways: at the network level, by configuring its Network Security Group (NSG) to allow all inbound traffic on all ports; and at the host level, by disabling the Windows Defender Firewall entirely. This meant the VM had effectively no filtering between it and the public internet, so any attacker scanning Azure's IP space could reach it directly.

### 1.2 Project Goals

- Stand up a disposable, isolated Azure environment dedicated to this lab
- Deploy a Windows VM configured as an internet-facing honeypot
- Confirm that attacker activity (failed logon attempts) is captured locally in Windows Event Logs
- Centralize that log data into a Log Analytics Workspace so it can be queried at scale
- Deploy Microsoft Sentinel as a SIEM layer on top of the centralized logs
- Learn and apply Kusto Query Language (KQL) to filter and analyze security events
- Enrich raw IP-address-only events with real-world geographic context using a Watchlist
- Build a live, visual attack map showing where attackers are connecting from

### 1.3 What I Did — Summary

- Provisioned an Azure subscription and dedicated resource group (`RG-SOC-Lab`) to contain every resource used in this lab
- Created a Virtual Network (`Vnet-soc-lab`) to host the honeypot's network interface
- Deployed a Windows 11 Pro virtual machine (`CORP-NET-EAST-1`, referred to as `Honey-Pot`), sized to comply with the subscription's VM-size policy restrictions
- Corrected a misconfigured NSG rule that had accidentally restricted inbound traffic to a single port, then opened all inbound ports/protocols so the VM would be reachable and attractive to attackers
- Connected over RDP and disabled the Windows Defender Firewall across all three profiles (Domain, Private, Public)
- Verified in Event Viewer that failed logon attempts (Event ID 4625) were being generated, both from intentional test logins and real, unsolicited attackers on the internet
- Created a Log Analytics Workspace (`soc-lab-law`) and deployed Microsoft Sentinel on top of it
- Installed the 'Windows Security Events' content solution from the Sentinel Content Hub
- Configured the 'Windows Security Events via AMA' data connector and created a Data Collection Rule (DCR) to stream the VM's Security event log into the workspace
- Queried the centralized logs with KQL, isolating Event ID 4625 (failed logon) records
- Imported a ~54,000-row geo-IP CSV dataset as a Sentinel Watchlist to map IP ranges to countries/cities
- Wrote an enrichment query using the `ipv4_lookup()` Kusto function to join failed logon events against the geo-IP watchlist
- Began building a Sentinel Workbook attack map to visualize enriched attack data geographically

### 1.4 Why This Project Matters

This lab mirrors real SOC workflows end-to-end: log source onboarding, centralized ingestion, KQL-based querying, data enrichment, and visualization — foundational, transferable skills for security operations, threat hunting, and detection engineering roles. Exposing a real VM to the internet also demonstrated just how immediate and constant background internet scanning and attack traffic actually is — within minutes of removing protection, real attackers began attempting to log in.

**Architecture:** Public Internet → NSG → Honeypot VM → Log Analytics Workspace → Microsoft Sentinel

## 2. Azure Environment Setup

A clean, isolated environment was prepared before deploying the honeypot itself, so the whole lab could be reasoned about and torn down completely by simply deleting one resource group.

- **Resource Group:** Created `RG-SOC-Lab` under the `XLabs-089` subscription, in the East US region.
- **Virtual Network:** Created `Vnet-soc-lab` inside `RG-SOC-Lab`, matching the resource group's region to avoid unnecessary cross-region latency or egress costs.

## 3. Deploying the Honeypot Virtual Machine

The VM was named `CORP-NET-EAST-1`, configured with a Windows 11 Pro (25H2, x64 Gen2) image, deployed to Availability Zone 1 in East US, using the 'Trusted launch virtual machines' security type.

> **Note on VM sizing:** This subscription enforces an Azure Policy restricting which VM sizes can be deployed. An initial attempt using `Standard_B2as_v2` failed with a policy violation; the error message listed approved sizes (`Standard_D2s_v3`, `Standard_B1s`, among others), and a compliant size was selected instead.

Once deployed, the resource group automatically contained several linked resources: a public IP address, the network security group, the network interface, the OS disk, and the VNet.

### 3.1 Exposing the VM to the Internet

By default, Azure NSGs only allow a minimal set of inbound traffic. For the VM to function as an effective honeypot — and even just to be reachable over RDP — its NSG needed reconfiguring to allow all inbound traffic on all ports and protocols.

**Initial misconfiguration:** The first custom rule, named 'Danger,' was meant to open all traffic but was mistakenly scoped to only port 8080. RDP (port 3389) was still blocked by the underlying `DenyAllInBound` rule, so Remote Desktop Connection attempts failed. This is a good example of a subtle but common misconfiguration: a rule can be named permissively and still fail to achieve its intended effect if the port range is too narrow. The fix was to edit the rule's destination port range from `8080` to a wildcard (`*`), covering every port including 3389.

After the fix, a broad Allow rule sat above the default `AllowVnetInBound`, `AllowAzureLoadBalancerInBound`, and `DenyAllInBound` rules. Because Azure NSGs evaluate rules in priority order, placing the permissive Allow rule at a lower priority number ensured it was evaluated — and matched — before the restrictive default Deny rule.

### 3.2 Connecting and Disabling the Host Firewall

With the network path open, RDP was used to log into the VM using its public IP address and the administrator credentials configured during VM creation. Once connected, `wf.msc` (Windows Defender Firewall with Advanced Security) was used to disable the local firewall across all three profiles: Domain, Private, and Public — removing the last layer of host-based filtering so the VM behaves as a genuinely unprotected target.

> **Security caution:** Disabling the host firewall and opening all inbound NSG rules intentionally maximizes attack surface. This configuration is appropriate only for a deliberately isolated, disposable lab VM built specifically to attract and study attacker behavior — never for a production system or any machine handling real data.

## 4. Capturing Failed Logon Events Locally

Before wiring up centralized log forwarding, it was important to confirm — directly on the VM — that expected security events were actually being generated. Windows records failed logon attempts as Event ID 4625 in the Security log. Event Viewer showed 728 security events, including multiple Audit Failure entries.

Notably, these failed logon events were generated both by intentional test attempts (using a fake username such as 'employee') and by real, unsolicited login attempts from external attackers scanning the internet — confirming the honeypot was already attracting genuine attacker traffic within a short window of exposure. Inspecting an individual 4625 event revealed the Logon Type (Type 3 = network logon, typical of RDP/SMB-based attempts), the attempted account name, the account domain, and the failure reason.

This local verification step matters because it establishes a ground truth: before any centralized pipeline is trusted, the underlying event source needs to be confirmed as actually producing the expected telemetry.

## 5. Centralizing Logs with Log Analytics & Microsoft Sentinel

Checking logs locally on a single machine doesn't scale, and isn't how real SOC teams operate. This phase centralized the honeypot's Security event log into a Log Analytics Workspace (LAW), then layered Microsoft Sentinel on top as a full SIEM.

- **5.1** Created the workspace `soc-lab-law` inside `RG-SOC-Lab`, East US, using the Pay-As-You-Go (Per GB 2018) pricing tier — worth monitoring in a lab where an internet-exposed VM may generate high, unpredictable event volume.
- **5.2** Deployed Microsoft Sentinel attached to `soc-lab-law`. Sentinel is not a separate resource with independent storage — it's a SIEM/SOAR layer that attaches to an existing workspace.
- **5.3** Installed the 'Windows Security Events' solution from Sentinel's Content Hub (480 available solutions), which provides two data connectors: 'Windows Security Events via AMA' (modern, Microsoft-recommended, using the Azure Monitor Agent) and 'Security Events via Legacy Agent' (being phased out). The AMA-based connector was the clear choice.
- **5.4** Created a Data Collection Rule (DCR) targeting the honeypot VM, specifying that Security Events should be collected. Creating this rule automatically triggers Azure to install the Azure Monitor Agent extension onto the target VM — the actual mechanism that streams Windows Event Log data into the workspace.
- **5.5** Once the DCR was active and the AMA extension finished installing, Security events began flowing into the workspace within minutes, confirmed via a basic `SecurityEvent` query tagged with the VM's hostname, `Honey-Pot`.

**Filtering for failed logons with KQL:**

```kql
SecurityEvent
| where EventID == 4625
| order by TimeGenerated desc
```

KQL is conceptually similar to SQL and to Splunk's SPL — all three are query languages built around filtering, projecting, and aggregating structured data, just with different syntax. Learning one transfers readily to the others, valuable since different organizations standardize on different platforms.

## 6. Log Enrichment with Geo-IP Data

Raw `SecurityEvent` records contain only an IP address for the source of a logon attempt — no country, city, or other geographic context. To turn that into something analysts can reason about visually, a Sentinel Watchlist was used to import a geo-IP mapping dataset of roughly 54,000 rows, each mapping a CIDR IP range to a geographic location.

The watchlist wizard was configured with Source type Local File, File type CSV with a header, 0 lines before the header row, and the `geoip-summarized.csv` dataset uploaded. The watchlist was given the alias `geoip`, with its Search Key set to the `network` column (containing CIDR-formatted IP ranges like `8.8.8.0/24`).

**Enrichment query**, joining failed logons against the geo-IP watchlist using Kusto's `ipv4_lookup()` function:

```kql
let GeoIPDB_FULL = _GetWatchlist("geoip");
let WindowsEvents = SecurityEvent
| where EventID == 4625
| order by TimeGenerated desc
| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);
WindowsEvents
```

This returns each failed logon event with additional columns — country, city, coordinates — derived from whichever CIDR block in the watchlist contains that event's source IP. In effect, it turns a table of anonymous IP addresses into a table with real geographic meaning, the foundation for the attack map built next.

## 7. Building the Live Attack Map

The final stage was visualizing the enriched, geo-located attack data on an interactive map inside a Sentinel Workbook — turning raw log rows into an at-a-glance picture of where attacks against the honeypot are originating from.

Inside the workbook, a 'data source + visualization' element was added, running the geo-IP enrichment query and rendering results on a map via the Advanced Editor using a JSON template that maps the query's latitude, longitude, and metric fields onto a world map. The resulting map plots each failed logon attempt as a point on a world map, giving an immediate, real-time sense of the global distribution of attacker traffic hitting the honeypot.

## 8. Summary & Key Takeaways

This project walked through the full lifecycle of a honeypot-driven SOC lab, from initial infrastructure setup through to a finished visual analytics product:

- Provisioned an isolated Azure resource group and virtual network to host the lab
- Deployed and intentionally exposed a Windows VM in Azure as an internet-facing honeypot, correcting an NSG misconfiguration along the way
- Disabled the host-level Windows Defender Firewall to remove all local filtering
- Verified attacker activity locally via Windows Event Viewer (Event ID 4625), confirming both test and real-world attacker traffic
- Centralized logs into a Log Analytics Workspace and Microsoft Sentinel using the Windows Security Events via AMA connector and a Data Collection Rule
- Queried and filtered security logs using Kusto Query Language (KQL)
- Enriched raw IP-only logs with geographic context via a ~54,000-row Sentinel Watchlist and `ipv4_lookup()`
- Began building a Sentinel Workbook attack map to visualize global attacker activity geographically

Taken together, this lab reinforced core SOC analyst and detection engineering skills: log source onboarding, SIEM query writing, data enrichment, and building visualizations that make raw telemetry actionable.
