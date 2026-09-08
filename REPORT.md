# SOC/SIEM Detection & Investigation Lab with Wazuh

Windows Security Monitoring, Detection Engineering, MITRE ATT&CK Mapping, and SOC Alert Investigation


---

## 1. Project Overview

This project documents the design and implementation of an isolated SOC/SIEM laboratory using Wazuh, Windows 11, Kali Linux, Windows Security auditing, PowerShell logging, and Sysmon.

The objective was to move beyond basic SIEM deployment by building and validating an end-to-end security monitoring workflow:

**Telemetry Generation → Log Collection → Detection → Alert Validation → Investigation → Event Correlation → Analyst Disposition**

Three controlled security scenarios were examined:

1. Local Windows account creation
2. PowerShell and registry-related activity
3. Failed Windows authentication

The failed-authentication scenario was taken through a complete SOC investigation. Windows Security Event ID 4625 was analyzed and correlated with a subsequent Event ID 4624 to determine whether the activity represented suspicious authentication behavior or expected user activity.

The project also involved troubleshooting several infrastructure and telemetry issues, including Wazuh Indexer memory exhaustion, OpenSearch startup timeouts, Filebeat-to-Indexer connectivity, Wazuh API availability, isolated-network connectivity, and a Sysmon EventChannel subscription issue.

Rather than treating every alert as malicious, the lab emphasizes evidence-based SOC analysis: validating what occurred, establishing context, correlating related events, and determining an appropriate disposition.

---

## 2. Project Objectives

The primary objectives of this project were to:

- Build an isolated SOC monitoring environment using virtual machines.
- Deploy and stabilize a Wazuh SIEM/XDR platform.
- Connect a Windows 11 endpoint to Wazuh using the Wazuh agent.
- Generate and collect Windows Security telemetry.
- Install Sysmon and validate enhanced endpoint telemetry locally.
- Configure PowerShell Operational and Script Block Logging.
- Generate controlled security events without using destructive payloads or malware.
- Validate the difference between raw event ingestion and SIEM alert generation.
- Investigate Wazuh alerts using Windows event fields and surrounding activity.
- Correlate failed and successful authentication events.
- Interpret Wazuh's built-in MITRE ATT&CK mappings while distinguishing those mappings from analyst conclusions.
- Troubleshoot failures across the endpoint, Wazuh Manager, Filebeat, Indexer, and Dashboard pipeline.
- Document remediation and verify the environment after changes.
- Perform post-lab cleanup and leave the environment in a stable state.

---

## 3. Lab Architecture and Network Design

The lab was built in VMware Workstation 17 Player using an isolated Host-only virtual network.

### Network

| Component | IP Address | Role |
|---|---|---|
| Kali Linux | `192.168.32.128` | Analyst workstation and Wazuh Dashboard access |
| Wazuh Manager | `192.168.32.130` | SIEM/XDR manager, indexer, dashboard, and alert processing |
| Windows-Target | `192.168.32.131` | Monitored Windows endpoint and controlled event-generation system |

**Host-only subnet:** `192.168.32.0/24`

### Architecture

```text
                     VMware Host
                         |
                Host-only Network
                  192.168.32.0/24
                         |
          +--------------+--------------+
          |                             |
     Kali Linux                   Wazuh Manager
   192.168.32.128                192.168.32.130
   Analyst System                 SIEM / Indexer
                                        |
                                  Wazuh Agent
                                        |
                                 Windows-Target
                                 192.168.32.131


---

## 4. Tools and Technologies

| Technology | Purpose |
|---|---|
| VMware Workstation 17 Player | Virtualization platform used to host the isolated SOC lab |
| Wazuh 4.14.7 | SIEM/XDR platform used for log collection, rule-based detection, alerting, and security monitoring |
| Wazuh Indexer | Stored and indexed security alert data for searching and analysis |
| Wazuh Dashboard | Used to perform Threat Hunting, review alerts, inspect event fields, and analyze MITRE ATT&CK mappings |
| Wazuh Agent 4.14.7 | Collected telemetry from the Windows endpoint and forwarded it to the Wazuh Manager |
| Windows 11 Pro | Monitored endpoint used to generate controlled security events |
| Windows Security Event Log | Primary telemetry source for authentication and account-management activity |
| Sysmon 15.21 | Installed to provide enhanced Windows endpoint telemetry and process-level visibility |
| PowerShell Operational Logging | Provided PowerShell execution telemetry |
| PowerShell Script Block Logging | Enabled additional visibility into executed PowerShell script blocks, including Event ID 4104 |
| Kali Linux | Analyst workstation used to access the Wazuh Dashboard and investigate alerts |
| MITRE ATT&CK | Framework used to contextualize Wazuh detection mappings |
| Filebeat | Forwarded Wazuh alert data to the Wazuh Indexer |

### Key Windows Event IDs Used

| Event ID | Meaning | Use in Lab |
|---|---|---|
| 4624 | Successful logon | Correlated with a preceding failed authentication |
| 4625 | Failed logon | Primary event for the failed-authentication detection and SOC investigation |
| 4672 | Special privileges assigned to a new logon | Observed during Windows Security telemetry analysis |
| 4688 | New process created | Verified locally as part of Windows process auditing |
| 4720 | User account created | Primary event for the account-creation detection |
| 4722 | User account enabled | Observed while investigating account-management activity |
| 4104 | PowerShell Script Block Logging | Used to validate PowerShell telemetry collection |
