# Wazuh SIEM Windows Endpoint Attack Detection Case Study

## Project Overview
This project is a **Windows endpoint attack detection case study** built using **Wazuh** and **Sysmon**.  
The goal of the lab was to simulate multiple suspicious activities on a Windows 10 endpoint, ingest the resulting telemetry into Wazuh, and validate whether the platform could detect, correlate, and expose those activities through its **Threat Hunting** interface.

Unlike a generic monitoring lab, this project was structured as a **hands-on SOC case study**:
- suspicious activity was manually generated on the Windows endpoint,
- Wazuh detections were reviewed,
- event details were investigated,
- and the alerts were mapped to likely attacker behaviors and MITRE ATT&CK techniques.

The case study focuses on **four major activity clusters**:
1. **Endpoint telemetry verification using Wazuh + Sysmon**
2. **Multiple failed Windows logon attempts**
3. **Registry Run Key persistence using `reg.exe`**
4. **Suspicious process / command shell execution involving `calc.exe`**

---

# Case Study Objective
The objective of this project was to answer the following SOC-style question:

> **If suspicious Windows endpoint activity is executed on a monitored host, can Wazuh + Sysmon capture enough telemetry to detect it, classify it, and support investigation?**

To answer this, the endpoint was instrumented with Wazuh and Sysmon, then several suspicious actions were performed and investigated through Wazuh.

---

# Environment Used

## Monitoring Stack
- **SIEM / XDR Platform:** Wazuh
- **Endpoint:** Windows 10
- **Telemetry Source:** Sysmon + Windows Security Event Logs
- **Attack / Support System:** Kali Linux
- **Shells Used:** Command Prompt and PowerShell

---

# Case Study Flow

This case study was executed in the following sequence:

1. **Verify Wazuh agent and Sysmon telemetry on the Windows endpoint**
2. **Generate suspicious Windows activity**
   - fake logon attempts
   - registry persistence
   - suspicious command execution
   - PowerShell / shortcut activity
3. **Search for detections in Wazuh Threat Hunting**
4. **Open event details and validate what Wazuh captured**
5. **Map findings to attacker behavior and MITRE ATT&CK**

---

# Phase 1 – Endpoint Telemetry Verification

Before generating any suspicious activity, the first step was to verify that the endpoint was properly instrumented.

## What was verified
- Wazuh service running on Windows
- Sysmon service running on Windows
- Recent Sysmon events visible from PowerShell
- Wazuh agent configuration present and pointing to the manager

## Screenshot Evidence
### 1) Wazuh Agent + Sysmon Verification
![Wazuh Agent and Sysmon Verification](01_wazuh_agent_sysmon_verification.jpg)

## Findings
The screenshot confirms that:
- the **Wazuh service** started successfully,
- **Sysmon** was installed and actively running,
- Sysmon event data could be retrieved from:
  `Microsoft-Windows-Sysmon/Operational`
- the Wazuh agent configuration file (`ossec.conf`) was present on the Windows host.

This phase is important because the rest of the case study depends on reliable collection of:
- **Windows Security logs**
- **Sysmon process creation events**
- **Sysmon registry modification events**

---

# Phase 2 – Suspicious Activity Generation on the Windows Endpoint

Once telemetry collection was verified, multiple suspicious actions were generated on the endpoint to test whether Wazuh would detect them.

---

# Scenario A – PowerShell Shortcut / LNK Activity

A PowerShell-based shortcut creation sequence was executed to generate suspicious command activity and produce endpoint telemetry associated with user execution / suspicious script behavior.

## Screenshot Evidence
### 2) PowerShell Shortcut / LNK Creation
![PowerShell LNK Creation](02_powershell_lnk_creation.jpg)

## What the screenshot shows
The screenshot shows PowerShell commands used to create a Windows shortcut (`.lnk`) object via `WScript.Shell`.  
The shortcut was configured to launch:

- **Target path:** `cmd.exe`
- **Arguments:** `/c calc.exe`

The initial command produced an error, after which the steps were executed line by line and the shortcut was saved successfully.

## Why this matters
From a SOC perspective, PowerShell-based shortcut creation is relevant because:
- it shows scripted user-execution behavior,
- it can be abused for phishing or persistence,
- and it produces endpoint telemetry useful for detection engineering.

This activity also complements later detections involving **cmd.exe** and **calc.exe**.

---

# Scenario B – Failed Windows Logon Attempts

The next activity was to generate failed authentication attempts using a fake user account.

## Command Used
The screenshot shows repeated use of the following command pattern:

```cmd
net use \\localhost /u:FakeHackerAccount Password123
