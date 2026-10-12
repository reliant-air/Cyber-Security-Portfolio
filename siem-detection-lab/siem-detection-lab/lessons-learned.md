# Lessons Learned

## 1. Wazuh alerts only on rule matches
**Symptom:** `rundll32.exe` and `schtasks.exe` ran successfully and were logged by Sysmon, but nothing showed in Wazuh Discover.

**Cause:** Wazuh indexes events into `wazuh-alerts-*` only when they match a rule. Unmatched raw events are dropped unless archives (`logall` / `logall_json` plus the Filebeat archives config) are enabled. The encoded PowerShell test appeared only because a built-in rule covered it.

**Decision:** Kept archives off (to avoid indexing every raw event in a small lab) and wrote targeted custom rules instead. Trade-off: archives would give broader hunting visibility; custom rules give cleaner, intentional alerts.

**Takeaway:** "No alert" does not mean "no telemetry." Always check the endpoint's own logs before concluding a technique is invisible.

## 2. A malformed `local_rules.xml` stops the manager
**Symptom:** After my first save, `wazuh-manager` failed to restart:
```
wazuh-analysisd: ERROR: (1226): Error reading XML file 'etc/rules/local_rules.xml'
wazuh-analysisd: CRITICAL: (1220): Error loading the rules
```

**Cause:** Malformed XML from a messy manual edit (leftover or duplicate tags).

**Fix:** Rewrote the file cleanly and restarted. Pasting was awkward because VirtualBox clipboard sharing was off by default; enabling Devices -> Shared Clipboard -> Bidirectional (or retyping in nano) resolved it.

**Takeaway:** Validate rule files before restarting production-style services. Keep a known-good copy of `local_rules.xml` (this repo's `rules/` folder serves that purpose).

## 3. Custom rule conventions
Custom rule IDs must be 100000 or higher; lower IDs are reserved for the built-in ruleset. `local_rules.xml` lives in `/var/ossec/etc/rules/` and survives Wazuh upgrades.

## 4. Prevention can hide detection
Defender blocked the certutil attempt before Sysmon logged a completed process creation, so the SIEM saw nothing. Layered defences need layered log sources.

## 5. Practical lab notes
- Windows Firewall blocks inbound ICMP by default; enable "File and Printer Sharing (Echo Request - ICMPv4-In)" for cross-VM ping.
- VirtualBox internal networks exclude the host; NAT port forwarding is needed for browser access to the dashboard.
- The all-in-one installer is sensitive to RAM, disk and connectivity; a failed run can leave a corrupted dpkg state, and reverting to a clean snapshot is often the fastest recovery.
- Ubuntu LVM may not use the full disk by default (`growpart`, `pvresize`, `lvextend`, `resize2fs`).
