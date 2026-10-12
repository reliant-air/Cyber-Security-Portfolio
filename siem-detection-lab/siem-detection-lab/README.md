# SIEM Detection Lab: Wazuh + Sysmon, Mapped to MITRE ATT&CK

A hands-on detection engineering lab built in VirtualBox. I simulated four common attacker techniques on a Windows Server endpoint, found where the default Wazuh ruleset left gaps, wrote custom rules to close them, and validated each one end to end.

## Summary

| | |
|---|---|
| **SIEM** | Wazuh 4.9 (all-in-one: indexer, manager, Filebeat, dashboard) |
| **Endpoint** | Windows Server 2022 Evaluation, Sysmon (SwiftOnSecurity config), Wazuh agent |
| **Platform** | VirtualBox, isolated internal network `siem-lab-net` (192.168.56.0/24) |
| **Techniques tested** | 4 (PowerShell, rundll32, scheduled tasks, certutil) |
| **Custom rules written** | 2 (IDs 100001, 100002), each validated with before/after evidence |
| **Key finding** | Wazuh only indexes events that match a rule, so two real techniques produced no alert until I wrote custom detections |

## Architecture

See [architecture/lab-architecture.md](architecture/lab-architecture.md).

```
Windows Server 2022 (192.168.56.10)            Ubuntu Server (192.168.56.11)
 Sysmon -> Event Log -> Wazuh agent  ------->  Wazuh manager -> indexer -> dashboard
 (WIN-SRV22)                                   (host access via NAT 8443 -> 443)
```

## Detection coverage

| Tactic | Technique | Simulated with | Data source | Detection | Result |
|---|---|---|---|---|---|
| Execution | [T1059.001](https://attack.mitre.org/techniques/T1059/001/) PowerShell | `powershell.exe -EncodedCommand` | Sysmon Event ID 1 | Default Wazuh rule | Detected out of the box |
| Persistence | [T1053.005](https://attack.mitre.org/techniques/T1053/005/) Scheduled Task | `schtasks /create ... /ru SYSTEM` | Sysmon Event ID 1 | **Custom rule 100001** (level 10) | Gap found, closed, validated |
| Defense Evasion | [T1218.011](https://attack.mitre.org/techniques/T1218/011/) Rundll32 | `rundll32.exe shell32.dll,Control_RunDLL` | Sysmon Event ID 1 | **Custom rule 100002** (level 8) | Gap found, closed, validated |
| Command and Control | [T1105](https://attack.mitre.org/techniques/T1105/) Ingress Tool Transfer / [T1140](https://attack.mitre.org/techniques/T1140/) | `certutil -urlcache -split -f` | Defender Operational log | Blocked by Windows Defender | Endpoint-layer prevention, not visible in Wazuh |

Per-technique write-ups are in [detections/](detections/). The rules are in [rules/local_rules.xml](rules/local_rules.xml).

## Method

1. **Baseline:** confirmed Sysmon events flowing from `WIN-SRV22` into Wazuh.
2. **Simulate:** ran each technique by hand, one command at a time, logging the timestamp and result.
3. **Observe:** checked Wazuh Discover and the local Windows logs to see what was and wasn't detected.
4. **Diagnose:** traced the missing alerts to Wazuh's rule-matching behaviour (see [lessons-learned.md](lessons-learned.md)).
5. **Detect:** wrote custom rules in `local_rules.xml`.
6. **Validate:** re-ran the same commands and confirmed the rules fired (`rule.id: 100001` and `rule.id: 100002`).

## Lessons learned and limitations

See [lessons-learned.md](lessons-learned.md). The most important limitation: the current rules match on process name only, so they are broad and would need tuning in a real environment. Each detection write-up lists specific false positives and evasion gaps.

## Roadmap (v2)

- Add a Windows 11 client and Active Directory.
- Detections for lateral movement and Kerberoasting.
- Tighten rules 100001 and 100002 with command-line matching.
- Ship the Defender Operational channel into Wazuh.

## Safety and scope

The lab is fully isolated (internal network only) and uses evaluation licences. All simulations were hand-run, benign test commands in my own environment.
