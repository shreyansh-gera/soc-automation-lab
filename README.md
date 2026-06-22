# SOC Automation Lab

A home lab simulating SOC operations with Wazuh SIEM, Sysmon endpoint monitoring,
and a custom real-time alerting pipeline to Discord.

## Architecture

| VM | Role | Interface | IP |    
| Ubuntu 24.04 | Wazuh SIEM + Custom Discord Integration | enp0s8 | 192.168.56.101 |
| Windows 11 | Victim endpoint, Wazuh Agent + Sysmon | Ethernet 2 | 192.168.56.102 |
| Kali Linux | Attacker | eth1 | 192.168.56.103 |

All three machines run on a VirtualBox host-only network, isolated from the internet.

## Demo

See `videos/brute-force-demo.mp4` for a live recording showing:
- Kali running Hydra SSH brute force
- Raw Wazuh alerts streaming in real time (`alerts.json`)
- Discord receiving formatted alerts with MITRE ATT&CK mapping

## Tools Used

- **Wazuh** — SIEM, log analysis, MITRE ATT&CK mapping
- **Sysmon** — Windows endpoint visibility
- **Python** — custom Wazuh-to-Discord integration
- **Kali Linux** — attack simulation (Hydra, Nmap, enum4linux)

## Attack Scenarios Detected

### 1. SSH Brute Force (MITRE T1110 — Credential Access)

Simulated a credential stuffing attack using Hydra against the Wazuh/Ubuntu server.

**Command used:**
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt 192.168.56.101 ssh -t 4
```

**Rules triggered:**
- Level 10 — Multiple authentication failures (Rule 40111)
- Level 10 — PAM: Multiple failed logins in a small period of time (Rule 5551)
- Level 8 — Maximum authentication attempts exceeded (Rule 5758)
- Level 5 — sshd: authentication failed (Rule 5760)

MITRE techniques mapped: **T1110.001** (Password Guessing), **T1021.004** (SSH / Lateral Movement)

### 2. Network & SMB Enumeration (MITRE T1135 — Network Share Discovery)

Performed reconnaissance against the Windows 11 victim to enumerate shares, users,
and workgroup information.

**Commands used:**
```bash
enum4linux -a 192.168.56.102
nmap --script smb-enum-shares,smb-enum-users -p 445 192.168.56.102
```

**Result:** Enumerated workgroup name, NetBIOS service info, and known usernames
on the target. Wazuh flagged related activity on the Windows11-Victim agent,
including a Level 15 critical alert for an executable dropped in a
malware-associated folder, and a Level 12 alert for event queue flooding caused
by the scan volume.

### 3. Process Injection False Positive Tuning

Identified and suppressed a recurring false positive (Rule 92910 — OneDrive
process legitimately accessing Explorer, flagged as possible process injection)
using a custom local rule. This demonstrates real-world alert tuning to reduce
analyst fatigue from noisy detections.

## How It Works

1. Wazuh manager monitors agent logs (Windows Sysmon + Ubuntu syslog/auth)
2. On a rule match at or above the configured severity level, a custom Python
   integration script executes
3. The script parses the alert JSON, extracting rule level, description, agent
   name, and MITRE ATT&CK tactic/technique
4. A formatted message is sent to Discord via webhook in real time

## Files

- `configs/ossec.conf` — Wazuh manager configuration
- `configs/custom-discord` — Python integration script (webhook redacted)
- `configs/local_rules.xml` — custom rule tuning (false positive suppression)

## Notes

During testing, the Wazuh dashboard UI had a persistent index-pattern filter
caching issue preventing the Threat Hunting view from displaying events, despite
alerts being correctly generated and indexed (verified via direct OpenSearch query
— 741+ alerts confirmed indexed). The underlying detection-to-alert pipeline
(manager → integration → Discord) operates independently of the dashboard UI
and was fully validated via `alerts.json` and live Discord delivery.

## Key Takeaways

- Deployed and configured Wazuh SIEM across multiple OS platforms
- Built a custom Python integration parsing security alerts to deliver
  real-time, MITRE-mapped notifications
- Simulated and validated detection of MITRE ATT&CK **T1110** (Brute Force) and
  **T1135** (Network Share Discovery)
- Performed alert tuning to reduce false positive noise
- Debugged real infrastructure issues, including Docker Swarm image-download
  timeouts, OpenSearch memory contention, and filesystem corruption recovery
  on the attacker VM
