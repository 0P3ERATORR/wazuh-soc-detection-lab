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
