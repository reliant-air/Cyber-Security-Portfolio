# Lab Architecture

## Hosts

| VM | OS | IP | Role |
|---|---|---|---|
| WIN-SRV22 | Windows Server 2022 Evaluation | 192.168.56.10 | Monitored endpoint: Sysmon + Wazuh agent |
| Wazuh server | Ubuntu Server | 192.168.56.11 | Wazuh all-in-one 4.9 (indexer, manager, Filebeat, dashboard) |

## Network

- Hypervisor: VirtualBox.
- VMs communicate over an internal network named `siem-lab-net` (192.168.56.0/24). Internal networks exclude the host by design.
- Dashboard access from the host browser uses NAT port forwarding: host `8443` -> guest `443`, then `https://localhost:8443`.

## Telemetry pipeline

1. **Sysmon** (SwiftOnSecurity config) writes to `Microsoft-Windows-Sysmon/Operational`.
2. The **Wazuh agent** reads that channel via a `<localfile>` block in `ossec.conf` (`eventchannel` format).
3. Events are sent to the **Wazuh manager**, which decodes them and evaluates them against the ruleset.
4. Events that **match a rule** become alerts and are indexed into `wazuh-alerts-*`.
5. Alerts are viewed in the **dashboard** (Discover / Threat Hunting).

The manager VM does not run a separate agent; the built-in agent 000 handles self-monitoring.

## Design decision

Archives (`logall` / `logall_json`) were left disabled. Indexing every raw event is useful for threat hunting but generates a lot of data in a small lab, so the lab relies on rule-matched alerts and closes gaps with custom rules instead.
