# SIEM Pipeline

**Sysmon and Splunk Enterprise | Windows Server 2025 Domain Controller | ADForest.local**

A fully operational security monitoring pipeline built on a Windows Server 2025 Active Directory Domain Controller. Sysmon captures host telemetry and forwards it natively into Splunk Enterprise, enabling real-time process monitoring, threat detection, and SOC analyst workflows — all within a 500 MB/day free licence constraint.

---

## Architecture Overview

```
[Windows Server 2025 DC]
        │
        ├── Sysmon64 (service)
        │     └── Microsoft-Windows-Sysmon/Operational (Windows Event Log)
        │
        └── Splunk Enterprise 10.2 (local indexer)
              ├── inputs.conf → WinEventLog stanza (renderXml=true)
              ├── props.conf  → sourcetype override (local/)
              └── Index: wineventlog
                    └── sourcetype: XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
```

Splunk Enterprise runs on the same host as the monitored system — no Universal Forwarder required. The built-in WinEventLog input monitor reads directly from the Sysmon operational log channel.

---

## Components

| Component | Version | Role |
|-----------|---------|------|
| Windows Server 2025 | DC | Host and AD domain controller |
| Sysmon64 | Latest | Host-based telemetry collection |
| Splunk Enterprise | 10.2.0 | Local SIEM indexer and search head |
| Sysmon TA (Splunk) | Community | Field extractions and CIM mapping |

---

## Sysmon Configuration

Event IDs monitored (whitelist):

| Event ID | Description |
|----------|-------------|
| 1 | Process Create |
| 8 | CreateRemoteThread |
| 11 | FileCreate |
| 12 | Registry object added/deleted |
| 13 | Registry value set |
| 22 | DNS Query |

Event IDs suppressed (blacklist):

| Event ID | Description | Reason |
|----------|-------------|--------|
| 3 | Network Connect | High volume — exceeds 500 MB/day licence |
| 10 | Process Access | Noisy on DC — high false positive rate |

---

## Splunk inputs.conf (Sysmon Stanza)

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
index = wineventlog
disabled = false
current_only = 1
start_from = oldest
renderXml = true
whitelist = 1,8,11,12,13,22
blacklist = 3,10
```

`renderXml = true` is required for the Sysmon TA to parse structured fields (Image, CommandLine, ParentImage, etc.) from XML-formatted events.

---

## Key Troubleshooting Resolved

### Sourcetype Rename Issue
The Sysmon TA ships with a `rename` directive in its default `props.conf` that reassigns the sourcetype away from the expected value. This was resolved by creating a `local/props.conf` override — the standard Splunk best-practice for TA configuration conflicts.

### Event ID 1 Silently Dropped
Splunk processes `blacklist` before `whitelist`. An initial configuration had Event ID 1 inadvertently covered by a blacklist pattern, causing process creation events to be silently dropped. Fixed by explicitly separating the whitelist and blacklist stanzas.

### Dashboard Sourcetype Mismatch
The SysmonProcessMonitor dashboard SPL contained a truncated sourcetype string (`xmlwineventlog`) that did not match the actual ingested sourcetype (`XmlWinEventLog:Microsoft-Windows-Sysmon/Operational`). Corrected directly in the dashboard data source editor.

---

## Dashboard: SysmonProcessMonitor

SPL query backing the process activity table:

```spl
index=wineventlog sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
| stats count by Image
| sort -count
| head 10
```

Displays the top 10 processes by Event ID 1 (Process Create) frequency — a primary indicator for lateral movement, LOLBins abuse, and persistence mechanisms.

---

## Evidence Screenshots

Screenshots are in the `/evidence/` folder:

| File | Description |
|------|-------------|
| `01_sysmon_service_running.jpg` | Sysmon64 service confirmed running in PowerShell |
| `02_sysmon_events_eventlog.jpg` | Event ID 1 events visible in Windows Event Log |
| `03_metrics_log_ingestion.jpg` | metrics.log confirming active Sysmon ingest (0.9 eps) |
| `04_splunk_sourcetypes.jpg` | All four sourcetypes confirmed in wineventlog index |
| `05_dashboard_populated.jpg` | SysmonProcessMonitor dashboard showing live process data |
| `06_inputs_conf.jpg` | inputs.conf Sysmon stanza configuration |

---

## Skills Demonstrated

- Sysmon deployment and operational log channel configuration
- Splunk Enterprise WinEventLog input configuration
- Sourcetype pipeline troubleshooting (props.conf override pattern)
- Ingest volume management within 500 MB/day licence constraints
- SPL query writing for process telemetry analysis
- Splunk dashboard creation and data source editing
- Systematic log-based diagnosis (splunkd.log, metrics.log)

---

## Related Projects

- [Active Directory Hardening + PingCastle](../AD-Hardening/)
- [BloodHound CE Deployment](../BloodHound/)
- [Suricata Network TAP (Topton N150)](../Suricata-TAP/)
- [Windows LAPS Implementation](../LAPS/)

---

*Part of a cybersecurity home lab portfolio targeting SOC Analyst and Security Administrator roles.*
