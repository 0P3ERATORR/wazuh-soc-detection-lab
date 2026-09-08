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
