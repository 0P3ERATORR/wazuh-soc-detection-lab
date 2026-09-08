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


### 8.3 Detection 3 — Failed Windows Authentication

A controlled failed authentication attempt was generated on Windows-Target by entering an incorrect password during an interactive Windows logon.

Windows generated:

- **Event ID:** 4625
- **Event:** An account failed to log on
- **Target account:** `0P3RAT0R`
- **Logon type:** 2 — Interactive logon

The event was successfully collected by the Wazuh agent and identified in the Wazuh raw event archives.

Wazuh generated an alert with the following details:

- **Wazuh Rule ID:** 60122
- **Rule level:** 5
- **Description:** Logon Failure - Unknown user or bad password
- **Rule group:** `authentication_failed`
- **MITRE ATT&CK ID:** T1531
- **Technique:** Account Access Removal
- **Tactic:** Impact

The detection chain was validated as:

```text
Controlled Failed Logon
        |
        v
Windows Security Event 4625
        |
        v
Wazuh Agent Collection
        |
        v
Wazuh Rule 60122
        |
        v
Level 5 Authentication Alert
```

### Alert Indexing Delay

During validation, the fresh Event ID 4625 was present in both the Wazuh raw archives and `alerts.json`, but initially did not appear in Threat Hunting.

Investigation of the alert pipeline identified repeated Filebeat connection failures to the Wazuh Indexer on TCP port 9200.

The Filebeat log showed connection-refused messages followed later by successful reconnection to the Indexer.

After connectivity was restored, the alert became searchable in the Wazuh Dashboard.

This demonstrated an important distinction between:

1. Event generation on the endpoint
2. Event collection by the Wazuh agent
3. Alert generation by the Wazuh Manager
4. Alert forwarding by Filebeat
5. Indexing and Dashboard search visibility

The event was then investigated further by correlating it with subsequent authentication activity.

> **MITRE ATT&CK note:** T1531 was the built-in mapping assigned by Wazuh Rule 60122. The presence of this mapping was not treated as proof that Account Access Removal had occurred. The final analyst disposition was based on the underlying Windows event fields and correlated activity.


---

## 9. SOC Alert Investigation — Failed Authentication

After validating the failed-authentication detection, the alert was investigated as a SOC analyst would investigate an authentication event in a monitored environment.

The objective was not to assume that the failed login was malicious, but to use the available evidence to determine what occurred and whether escalation was warranted.

### Investigation Question 1 — Which account was targeted?

The following Windows event field was examined:

`data.win.eventdata.targetUserName`

**Finding:**

`0P3RAT0R`

This identified the Windows account involved in the failed authentication attempt.

### Investigation Question 2 — What type of logon was attempted?

The following field was examined:

`data.win.eventdata.logonType`

**Finding:**

`2`

Windows Logon Type 2 represents an interactive logon, indicating that the authentication attempt occurred through an interactive Windows session rather than a remote network logon.

### Investigation Question 3 — Where did the attempt originate?

The source address recorded in the event was examined:

`data.win.eventdata.ipAddress`

**Finding:**

`127.0.0.1`

The localhost address supported the conclusion that the authentication attempt originated from the monitored Windows endpoint itself rather than from a remote system.

### Investigation Question 4 — Why did authentication fail?

The failure-reason field was examined:

`data.win.eventdata.failureReason`

**Finding:**

`%%2313`

The raw value was retained as observed in the Wazuh event data and considered together with the Windows authentication status codes examined next.

### Investigation Question 5 — What was the authentication status code?

The following field was examined:

`data.win.eventdata.status`

**Finding:**

`0xC000006D`

This status indicates that the logon attempt failed because the supplied credentials were invalid.

### Investigation Question 6 — What did the SubStatus reveal?

The following field was examined:

`data.win.eventdata.subStatus`

**Finding:**

`0xC000006A`

This provided additional context indicating that the username was valid but the password supplied was incorrect.

### Investigation Question 7 — Was the activity isolated or repeated?

Wazuh Rule 60122 was reviewed to identify other failed-authentication alerts.

Three relevant hits were observed:

- September 7, 2026 — controlled failed-authentication event
- September 6, 2026 — two earlier failed-authentication events approximately three seconds apart

The two earlier events were noteworthy, but their presence alone was not sufficient to classify the activity as brute force or password spraying.

Around the September 7 controlled event, no cluster of repeated failures was identified.

### Investigation Question 8 — Was there a successful login afterward?

The investigation was expanded beyond failed-authentication alerts to search for Windows Security Event ID 4624 — Successful Logon.

A successful authentication for the same account, `0P3RAT0R`, was identified approximately 12 seconds after the failed Event ID 4625.

This correlation provided important context for determining the nature of the failed authentication.

### Investigation Question 9 — Was the successful login also interactive?

The Event ID 4624 logon type was examined.

**Finding:**

`2`

The successful authentication was also an interactive logon.

The sequence therefore showed:

```text
Failed Interactive Logon — Event 4625
             |
             | ~12 seconds
             v
Successful Interactive Logon — Event 4624
             |
             v
Same account: 0P3RAT0R
```

### Investigation Question 10 — What was the analyst disposition?

Based on the available evidence, the activity was assessed as **likely benign / expected user activity**.

The investigation established that:

- A single interactive authentication failure occurred for `0P3RAT0R`.
- The SubStatus `0xC000006A` indicated an incorrect password for a valid account.
- The event originated from localhost.
- A successful interactive logon for the same account followed approximately 12 seconds later.
- No burst of repeated failures was identified around the controlled September 7 event.

The sequence was therefore consistent with a user entering an incorrect password and correcting it on the next attempt rather than with brute-force or password-spraying activity.

In a production environment, additional escalation would be appropriate if the event formed part of repeated authentication failures, involved an unusual source, targeted a privileged account, originated from an unexpected endpoint, or correlated with other suspicious activity.

### Investigation Conclusion

This investigation demonstrated why SOC analysis requires more than reading an alert description or MITRE ATT&CK mapping.

By examining the underlying event fields and correlating Event ID 4625 with the subsequent Event ID 4624, the alert could be placed into context and assigned an evidence-based disposition rather than automatically being classified as malicious.


---

## 10. MITRE ATT&CK Mapping

MITRE ATT&CK mappings provided by Wazuh were reviewed during the detection scenarios to understand how observed activity related to recognized adversary techniques.

The mappings were treated as contextual information rather than automatic confirmation of malicious activity.

| Detection / Activity | Wazuh Rule | MITRE ID | Technique | Tactic |
|---|---:|---|---|---|
| Local account creation | 60109 | T1098 | Account Manipulation | Persistence |
| PowerShell execution | Built-in PowerShell detection | T1059.001 | PowerShell | Execution |
| Registry modification through PowerShell | Built-in PowerShell detection | T1112 | Modify Registry | Defense Evasion |
| Failed authentication | 60122 | T1531 | Account Access Removal | Impact |

### Analyst Interpretation

The account-creation scenario provided a clear example of ATT&CK contextualization. Windows Event ID 4720 triggered Wazuh Rule 60109 and was mapped to **T1098 — Account Manipulation** under the Persistence tactic.

The PowerShell scenario demonstrated that ATT&CK-mapped behavior is not automatically malicious. The registry modification was performed intentionally to enable Script Block Logging, yet the behavior still matched techniques associated with PowerShell execution and registry modification.

Similarly, Wazuh Rule 60122 mapped the failed authentication event to **T1531 — Account Access Removal**. This was recorded as Wazuh's built-in mapping, but the investigation did not conclude that Account Access Removal had occurred.

Correlation of the failed Event ID 4625 with a successful Event ID 4624 approximately 12 seconds later supported a benign mistyped-password explanation.

This distinction is important in SOC operations:

**MITRE ATT&CK describes adversary behaviors and provides useful detection context, but analyst investigation determines whether the observed activity is actually suspicious or malicious.**


---

## 11. Troubleshooting and Root-Cause Analysis

Several technical issues occurred during the project. Rather than removing these from the final documentation, they were retained because they demonstrate troubleshooting across multiple layers of the SIEM pipeline.

### 11.1 Wazuh Indexer Out-of-Memory Failure

**Symptom:**  
The Wazuh Dashboard became unavailable and initially appeared to have an authentication problem.

**Investigation:**  
Service status, Indexer logs, memory utilization, JVM configuration, disk capacity, and Linux kernel logs were examined.

Kernel logs confirmed that the Linux OOM killer had terminated the OpenSearch Java process.

The investigation identified:

- Approximately 5.6 GiB usable VM memory
- Approximately 2.9 GB OpenSearch JVM heap
- No configured swap space
- Sufficient disk capacity

**Root cause:**  
The VM experienced memory pressure while running the Wazuh stack. OpenSearch also requires memory outside its configured JVM heap, and other Wazuh and operating-system processes were competing for the available physical memory. With no swap configured, the kernel eventually invoked the OOM killer.

**Remediation:**  
A persistent 4 GiB swap file was configured.

**Verification:**  
Indexer availability was subsequently confirmed through service status, TCP port 9200, authenticated Indexer access, and successful Dashboard login.

### 11.2 Wazuh Indexer Startup Timeout

**Symptom:**  
On some VM startups, the Indexer remained in an initializing state long enough for systemd to terminate it.

**Investigation:**  
The Indexer systemd startup timeout was found to be three minutes.

**Root cause:**  
On the resource-constrained lab host, OpenSearch could occasionally require longer than the configured timeout to initialize.

**Remediation:**  
A persistent systemd override increased:

```text
TimeoutStartSec=3min
```

to:

```text
TimeoutStartSec=10min
```

**Verification:**  
On subsequent startup, the Indexer was allowed additional initialization time and successfully reached an active state.

### 11.3 Wazuh Manager and API Availability

**Symptom:**  
The Dashboard reported API connectivity problems, including messages indicating that no API was available.

**Investigation:**  
The Wazuh Manager service, API listener on TCP port 55000, Dashboard API configuration, and API authentication were checked.

The API listener was reachable, and authenticated API access was successfully validated.

**Remediation:**  
The affected Wazuh services were restarted after confirming the Indexer and Manager dependencies were available.

**Verification:**  
Dashboard access and API communication were restored.

### 11.4 Kali Host-Only Network Connectivity

**Symptom:**  
Kali Linux temporarily lost connectivity to the Wazuh Dashboard.

**Investigation:**  
The Kali network interface was found to be down and initially lacked its expected IPv4 address.

**Remediation:**  
The interface was brought back online and the lab address `192.168.32.128/24` was restored.

**Verification:**  
Kali subsequently regained access to the Host-only lab network and Wazuh Dashboard.

### 11.5 Sysmon EventChannel Subscription Issue

**Symptom:**  
Sysmon was installed successfully and generated events locally, but the Wazuh Windows agent returned:

```text
ERROR: Could not EvtSubscribe() for (Microsoft-Windows-Sysmon/Operational) which returned (15007)
```

**Investigation:**  
The Sysmon Operational channel was verified to exist, was enabled, and contained valid events. Direct queries against the channel succeeded. The Wazuh agent service was also confirmed to be running under LocalSystem.

**Finding:**  
Despite the valid local channel, the Wazuh agent continued to return EventChannel subscription error 15007.

**Disposition:**  
The issue was documented as an unresolved Wazuh/Sysmon EventChannel subscription or compatibility issue. Successful central Sysmon ingestion was not claimed in the project results.

### 11.6 Raw Event Collection vs Alert Generation

**Symptom:**  
Some Windows events were known to exist on the endpoint but were not immediately visible as alerts in Threat Hunting.

**Investigation:**  
Wazuh raw event archiving was temporarily enabled using `logall_json` so that collected events could be inspected independently of SIEM rule matches.

This allowed the monitoring pipeline to be separated into two questions:

1. Did Wazuh collect the event?
2. Did a Wazuh rule generate an alert from the event?

**Finding:**  
Raw archive inspection proved that telemetry could be successfully collected even when a corresponding Threat Hunting alert was absent.

**Cleanup:**  
After validation was complete, `logall_json` was returned to `no` to avoid unnecessary long-term raw event storage.

### 11.7 Filebeat-to-Indexer Connectivity and Delayed Alert Visibility

**Symptom:**  
The controlled Event ID 4625 was present in the Wazuh raw archives and `alerts.json`, but the new alert initially did not appear in Threat Hunting.

**Investigation:**  
Filebeat output connectivity was tested and its logs were examined.

The logs showed repeated connection failures to:

```text
https://127.0.0.1:9200
```

followed later by successful reconnection to the Wazuh Indexer.

**Root cause:**  
The Wazuh Manager had successfully generated the alert, but Filebeat temporarily could not forward it to the Indexer. This delayed its availability in Dashboard searches.

**Verification:**  
After Filebeat re-established its Indexer connection, the Event ID 4625 alert became searchable in Threat Hunting.

This troubleshooting exercise demonstrated the importance of understanding the complete SIEM data path:

```text
Endpoint
   |
   v
Wazuh Agent
   |
   v
Wazuh Manager
   |
   +----> Raw Events
   |
   +----> Alert Generation
              |
              v
           Filebeat
              |
              v
        Wazuh Indexer
              |
              v
       Dashboard / Search
```

A failure at one stage does not necessarily mean that the preceding stages have also failed.


---

## 12. Verification and Validation

Final validation was performed to confirm that the monitoring environment remained operational after the detection exercises and troubleshooting activities.

### 12.1 Wazuh Platform Health

The status of the three primary Wazuh services was checked:

```bash
systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard
```

All three services returned:

```text
active
```

This confirmed that the Wazuh Indexer, Manager, and Dashboard were operational at the conclusion of the lab.

### 12.2 Windows Monitoring Services

The Windows endpoint was checked to confirm that the Wazuh agent and Sysmon services remained operational.

```powershell
Get-Service WazuhSvc,Sysmon64 | Select-Object Name,Status
```

Both services returned a status of:

```text
Running
```

### 12.3 Detection Validation

The completed scenarios demonstrated successful detection and investigation of:

- Windows local account creation through Event ID 4720 and Wazuh Rule 60109.
- PowerShell Operational and Event ID 4104 telemetry collection, including a Wazuh alert associated with PowerShell registry modification.
- Failed Windows authentication through Event ID 4625 and Wazuh Rule 60122.
- Correlation of the failed authentication with a subsequent successful Event ID 4624.

Raw event inspection was also used where necessary to distinguish successful telemetry collection from SIEM alert generation.

### 12.4 Final Environment State

At the end of validation:

- Wazuh core services were active.
- The Windows Wazuh agent was running.
- Sysmon remained installed and running locally.
- Windows Security telemetry was successfully reaching Wazuh.
- PowerShell Operational telemetry was successfully reaching Wazuh.
- Controlled test accounts were removed.
- Temporary raw event archiving was disabled.
- The lab VMs were shut down cleanly.

These checks provided a final verification that the environment remained stable after the monitoring and investigation exercises.


---

## 13. Cleanup and Environment Restoration

Post-lab cleanup was performed after the detection scenarios and investigations were completed.

### 13.1 Removal of Test Accounts

The temporary Windows accounts created during the account-management detection exercises were removed:

- `Soc-TestUser`
- `SOC-TestUser2`

The legitimate lab account `0P3RAT0R` was preserved.

The local user list was reviewed afterward to verify that the temporary accounts were no longer present.

### 13.2 Raw Event Archiving

Wazuh raw JSON event archiving had been temporarily enabled during troubleshooting to determine whether Windows events were being collected independently of alert generation.

After the required evidence had been obtained, the configuration was restored to:

```xml
<logall_json>no</logall_json>
```

The Wazuh Manager was restarted and confirmed to be active after the configuration change.

### 13.3 Service Verification

Before shutdown, the Wazuh platform was checked and the following services were confirmed active:

- Wazuh Indexer
- Wazuh Manager
- Wazuh Dashboard

On Windows-Target, both the Wazuh agent and Sysmon services were confirmed to be running.

### 13.4 Controlled Shutdown

The virtual machines were shut down cleanly after final validation.

The shutdown order used was:

1. Windows-Target
2. Kali Linux
3. Wazuh Manager

This left the SOC lab in a clean and recoverable state for future use.


---

## 14. Key Findings and Lessons Learned

This project demonstrated that deploying a SIEM is only the beginning of an effective monitoring workflow. Reliable detection requires visibility and validation across the entire telemetry pipeline.

### Key Findings

- Windows Security auditing provided reliable telemetry for account-management and authentication monitoring.
- Windows Event ID 4720 successfully produced a Wazuh account-creation alert through Rule 60109.
- Windows Event ID 4625 successfully produced a failed-authentication alert through Rule 60122.
- PowerShell Operational and Script Block telemetry could be centrally collected and inspected through Wazuh.
- Raw event collection and SIEM alert generation are separate stages and should be validated independently.
- An alert can be generated successfully by the Wazuh Manager while remaining temporarily unavailable in Threat Hunting if downstream indexing is interrupted.
- MITRE ATT&CK mappings provide useful context but should not replace analyst investigation.
- Correlation with surrounding events can significantly change the interpretation of an alert.

### Lessons Learned

One of the most important lessons from the lab was to troubleshoot security monitoring as a pipeline rather than treating the Dashboard as the entire SIEM.

The effective troubleshooting model became:

```text
Event Generation
      |
      v
Endpoint Logging
      |
      v
Agent Collection
      |
      v
Manager Processing
      |
      v
Rule / Alert Generation
      |
      v
Forwarding
      |
      v
Indexing
      |
      v
Dashboard Search
```

When an event was missing from the Dashboard, each stage could therefore be tested independently.

The failed-authentication investigation also reinforced the importance of evidence-based analysis. Event ID 4625 initially represented a failed authentication alert, but examination of the status codes, source, logon type, event frequency, and subsequent Event ID 4624 provided enough context to classify the controlled event as likely benign.

The project also provided practical experience troubleshooting infrastructure problems that can affect security monitoring, including memory exhaustion, service startup timing, API availability, network connectivity, EventChannel subscriptions, and SIEM indexing delays.

Overall, the lab strengthened both technical SIEM troubleshooting skills and the analytical process required to determine what a security alert actually means.
