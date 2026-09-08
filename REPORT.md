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


---

## 5. Wazuh Deployment and Platform Stabilization

### 5.1 Wazuh Deployment

Wazuh 4.14.7 was deployed as a virtual appliance and configured as the central SIEM/XDR platform for the lab.

The Wazuh virtual machine was configured with:

- 6 GB RAM
- 2 virtual CPUs
- Host-only networking
- Static lab IP address: `192.168.32.130`

The deployment provided the core components required for the monitoring environment:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

The Dashboard was accessed from the Kali Linux analyst workstation using HTTPS.

### 5.2 Wazuh Indexer Memory Failure

During deployment, the Wazuh Dashboard became unavailable. Initial symptoms appeared to indicate a Dashboard authentication problem; however, further investigation showed that the underlying Wazuh Indexer service was not running correctly.

Service and kernel logs were examined to identify the root cause.

The Linux kernel reported that the OpenSearch Java process had been terminated by the Out-of-Memory (OOM) killer.

Additional investigation identified the following conditions:

- Approximately 5.6 GiB of usable memory was available to the Wazuh VM.
- The OpenSearch JVM was configured with approximately 2.9 GB of heap memory.
- No swap space was configured.
- Disk capacity was not the cause of the failure.

This demonstrated that the apparent Dashboard problem was actually an underlying resource-availability issue affecting the Indexer.

### 5.3 Persistent Swap Remediation

To provide additional memory protection during periods of high memory pressure, a persistent 4 GiB swap file was created and enabled.

The remediation included:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Swap availability was verified using:

```bash
free -h
swapon --show
```

The swap file was then added to `/etc/fstab` so that it would remain available after reboot.

After remediation, the Wazuh Indexer was restarted and its availability was validated through service status, TCP port 9200, authenticated Indexer access, and successful Wazuh Dashboard login.

### 5.4 Indexer Startup Timeout

A separate reliability issue was later identified during VM startup. The Wazuh Indexer occasionally required longer than the default systemd startup timeout and was terminated before initialization completed.

The configured startup timeout was found to be three minutes.

A persistent systemd override was created to increase the Wazuh Indexer startup timeout to ten minutes:

```ini
[Service]
TimeoutStartSec=10min
```

After reloading systemd and restarting the environment, the Indexer was allowed sufficient time to initialize and subsequently reached an active state.

This change improved startup reliability on the resource-constrained lab host.

### 5.5 Verification

Platform health was validated by confirming that the three primary Wazuh services were active:

```bash
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard
```

The final health check returned all three services as active.

These troubleshooting steps reinforced the importance of investigating the complete SIEM pipeline rather than assuming that a visible Dashboard or authentication error originates at the user-interface layer.


---

## 6. Windows Endpoint and Wazuh Agent Deployment

### 6.1 Windows 11 Target

A Windows 11 Pro virtual machine named `Windows-Target` was deployed as the monitored endpoint.

The endpoint was configured on the isolated Host-only network with the following lab address:

`192.168.32.131`

The system served as the primary source of Windows Security, PowerShell, and Sysmon telemetry throughout the project.

### 6.2 Wazuh Agent Installation

Wazuh Agent 4.14.7 was installed on Windows-Target and configured to communicate with the Wazuh Manager at:

`192.168.32.130`

After installation, the Wazuh service was started and connectivity to the Manager was validated through the Wazuh Dashboard.

The endpoint successfully registered with the following details:

- Agent name: `Windows-Target`
- Agent ID: `001`
- IP address: `192.168.32.131`
- Operating system: Windows 11 Pro
- Agent version: Wazuh 4.14.7
- Status: Active

### 6.3 Agent Verification

Successful registration demonstrated that the basic endpoint-to-SIEM communication path was operational:

```text
Windows-Target
      |
      | Wazuh Agent
      v
Wazuh Manager
      |
      v
Indexer / Dashboard
```

The active-agent status in the Wazuh Dashboard was retained as evidence of successful endpoint integration.

This established the telemetry pipeline required for the subsequent Windows Security monitoring, PowerShell logging, controlled detection scenarios, and SOC investigations.


---

## 7. Sysmon and Windows Telemetry Configuration

### 7.1 Sysmon Installation

Sysmon 15.21 from Microsoft Sysinternals was installed on Windows-Target to provide enhanced endpoint telemetry.

After installation, the Sysmon service and driver were confirmed to be running.

Local event generation was validated through the Windows event channel:

`Microsoft-Windows-Sysmon/Operational`

Sysmon events, including process creation activity, were successfully observed locally. This confirmed that Sysmon itself was installed and functioning correctly on the Windows endpoint.

### 7.2 Wazuh Sysmon Collection Configuration

The Windows Wazuh agent configuration was updated to collect the Sysmon Operational event channel:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

After restarting the Wazuh agent, the agent reported the following EventChannel subscription error:

```text
ERROR: Could not EvtSubscribe() for (Microsoft-Windows-Sysmon/Operational) which returned (15007)
```

Windows error `15007` corresponds to an EventChannel-not-found condition from the subscription attempt.

Further validation showed that the Sysmon channel did exist, was enabled, and contained events. The channel could be queried successfully using Windows Event Log utilities.

The Wazuh service was also confirmed to be running under the LocalSystem account.

Because local Sysmon event generation was verified while the Wazuh agent continued to return the subscription error, the issue was documented as an unresolved EventChannel subscription/compatibility issue rather than incorrectly reporting successful central Sysmon ingestion.

### 7.3 Windows Security Telemetry

Windows Security auditing provided the primary telemetry used for the successful detection scenarios.

Events observed during the project included:

- Event ID 4624 — Successful logon
- Event ID 4625 — Failed logon
- Event ID 4672 — Special privileges assigned to a new logon
- Event ID 4688 — New process created
- Event ID 4720 — User account created
- Event ID 4722 — User account enabled

Windows Security events were successfully collected by the Wazuh agent and processed by the Wazuh Manager.

This telemetry became the foundation for the account-creation detection and the failed-authentication investigation performed later in the project.


---

## 8. Detection Engineering and Controlled Scenarios

Controlled activity was generated on Windows-Target to validate the monitoring pipeline and examine how endpoint events progressed from Windows telemetry to Wazuh detections.

The scenarios were intentionally non-destructive and focused on common activity that a SOC analyst may encounter during monitoring.

### 8.1 Detection 1 — Local Windows Account Creation

A controlled local user account was created on Windows-Target to generate Windows account-management telemetry.

Windows recorded the activity as:

- **Event ID:** 4720
- **Event:** A user account was created
- **Target account:** `SOC-TestUser2`

The event was successfully collected by the Wazuh agent and confirmed in the Wazuh raw event archives.

Wazuh subsequently generated an alert with:

- **Wazuh Rule ID:** 60109
- **Rule level:** 8
- **Description:** User account enabled or created
- **MITRE ATT&CK ID:** T1098
- **Technique:** Account Manipulation
- **Tactic:** Persistence

The detection chain was therefore validated as:

```text
Controlled Account Creation
        |
        v
Windows Security Event 4720
        |
        v
Wazuh Agent Collection
        |
        v
Wazuh Rule 60109
        |
        v
Level 8 Alert
        |
        v
MITRE ATT&CK T1098
Account Manipulation
```

This scenario demonstrated the distinction between simply collecting a Windows event and generating a SIEM alert from that event.

The test account was removed during post-lab cleanup after the detection and supporting evidence had been validated.


### 8.2 Detection 2 — PowerShell and Registry Activity

PowerShell Operational logging was configured as an additional telemetry source on Windows-Target.

The Wazuh agent configuration was updated to collect:

`Microsoft-Windows-PowerShell/Operational`

After restarting the agent, PowerShell Operational events were successfully observed in the Wazuh raw event archives.

### Script Block Logging

PowerShell Script Block Logging was enabled through the Windows registry to increase visibility into executed PowerShell commands.

A controlled PowerShell command was then executed, and Windows Event ID 4104 was successfully generated locally.

The corresponding Event ID 4104 telemetry was also identified in the Wazuh raw archives, confirming successful central collection of PowerShell Script Block events.

During this configuration activity, Wazuh generated an alert for the PowerShell command that modified the registry to enable Script Block Logging.

The alert included Wazuh's built-in MITRE ATT&CK mappings:

- **T1059.001 — PowerShell**
- **T1112 — Modify Registry**
- **Tactics observed:** Execution and Defense Evasion

It is important to distinguish the SIEM detection from analyst interpretation. In this case, the registry modification was intentionally performed as part of the lab configuration and was therefore benign.

This scenario demonstrated that a security tool may correctly detect behavior associated with ATT&CK techniques even when the underlying activity is authorized.

The analyst's responsibility is therefore not simply to treat a MITRE-mapped alert as malicious, but to establish context and determine whether the activity is expected or suspicious.
