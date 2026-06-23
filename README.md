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
```
## 3) Failed Logon Attempts via Command Prompt

What the screenshot shows

The command prompt output shows repeated failed attempts using the username:

FakeHackerAccount

The Windows system returns:

System error 1326
The user name or password is incorrect.

## Scenario C – Registry Run Key Persistence

A persistence mechanism was then created by adding a malicious autorun registry entry under the current user Run key.

Command Used
```bash
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v "MaliciousUpdate" /t REG_SZ /d "C:\Windows\System32\calc.exe" /f
```
## 4) Registry Persistence Creation via reg.exe

The command prompt shows successful creation of a registry value named:

MaliciousUpdate

pointing to:
```bash
C:\Windows\System32\calc.exe
```
under the current user Run path.

##Scenario D – Kali Reconnaissance Activity

A Kali Linux host was also used in the lab to perform an aggressive Nmap scan against a Windows system.

Screenshot Evidence
## 15) Kali Nmap Scan Terminal



The Kali terminal shows an Nmap aggressive scan (-A) against:

192.168.0.108

The scan output identifies:

open ports,
service details,
OS fingerprinting information,
and network distance.
Phase 3 – Wazuh Threat Hunting Review

After generating the suspicious activities, the next step was to investigate the endpoint in Wazuh Threat Hunting and validate which detections appeared.

Wazuh Detection Overview
Screenshot Evidence
## 5) Sysmon Events Overview in Wazuh

This screenshot provides the overall Wazuh event view for the Windows endpoint and shows that the platform captured multiple relevant detections, including:

Suspicious Windows cmd shell execution
Windows command prompt started by an abnormal process
PowerShell execution
Executable dropped in a folder commonly used by malware
other Sysmon-based detections

## Phase 4 – Failed Logon Investigation

The failed logon scenario was then investigated in detail.

## 6) Multiple Logon Failures Detection

What this screenshot shows

Wazuh detected repeated authentication failures and surfaced them as a higher-level alert:

Multiple Windows Logon Failures

This indicates that Wazuh did not merely store isolated failed logon events — it recognized the repeated pattern.

## 7) Failed Logon Event Details

Key evidence visible in the event details

The event details show:

target username: FakeHackerAccount
Windows Security Event ID: 4625
failed logon metadata such as:
logon type
status / substatus codes
target user fields
workstation information
Detection interpretation

This confirms that the repeated net use commands successfully generated Windows Security logon failure events and that Wazuh could ingest and expose them for investigation.

## Phase 5 – Registry Persistence Investigation

The registry persistence activity was then validated inside Wazuh.

Screenshot Evidence
## 8) Registry Run Key Detection

What this screenshot shows

Wazuh detected the registry modification and raised an alert describing:

Registry entry to be executed on next logon was modified using command line application reg.exe

This confirms that the persistence action was not only logged by Sysmon, but also interpreted by Wazuh as a suspicious Run Key modification.

## 9) Registry Run Key Event Details

Key evidence visible in the event details

The detailed event view shows:

Sysmon Event ID 13 (registry value set / registry modification)
Image: C:\Windows\system32\reg.exe
TargetObject: path ending in
\Software\Microsoft\Windows\CurrentVersion\Run\MaliciousUpdate
Details: C:\Windows\System32\calc.exe
Detection interpretation

This proves that the exact persistence action performed from the command line was captured at the telemetry level and surfaced with the key forensic fields needed for investigation:

the process that made the change,
the registry object modified,
and the value written.

## Phase 6 – Suspicious Registry Value / Secondary Detection

In addition to the Run Key alert, Wazuh also surfaced another detection tied to the registry value.

Screenshot Evidence
## 10) Base64-like Registry Value Detection

What this screenshot shows

The Wazuh alert describes:

Value added to registry key has Base64-like pattern

Although the registry value in this lab pointed to calc.exe, the screenshot demonstrates that Wazuh has content-based registry detection logic capable of flagging suspicious values based on pattern analysis.

Why this matters

This is useful from a SOC perspective because registry monitoring is not limited to where a value was written — it can also include logic about what the value looks like.

## Phase 7 – Suspicious Process / Command Execution Investigation

The next part of the case study focused on the execution of calc.exe and related suspicious command shell activity.

A. Calc Process Detection
Screenshot Evidence
## 11) Calc Process Detection

What this screenshot shows

The Wazuh event view shows a detection associated with:

data.win.eventdata.image:*calc.exe*

The rule description visible in the screenshot is:

Suspicious Windows cmd shell execution

This indicates that the calc.exe execution was visible in the monitored telemetry and linked to suspicious shell behavior.
## 12) Calc Process Event Details

Key evidence visible in the event details

The detailed Sysmon event includes:

Image: C:\Windows\System32\calc.exe
Original filename: CALC.EXE
Description: Windows Calculator
CommandLine: calc.exe
Detection interpretation

This confirms that Sysmon process creation logging was working correctly and that the execution of calc.exe could be traced in detail through Wazuh.

B. Suspicious CMD Shell Detection
Screenshot Evidence
13) Suspicious CMD Execution Detection

What this screenshot shows

Wazuh flagged a detection with the rule description:

Suspicious Windows cmd shell execution

The event list associates this activity with the monitored Windows endpoint and places it within the same time window as the other suspicious actions.

Screenshot Evidence
## 14) Suspicious CMD Execution Event Details

Key evidence visible in the event details

The detailed event panel shows:

- Sysmon-based telemetry
- Wazuh rule metadata
- Rule description: Suspicious Windows cmd shell execution
- MITRE ATT&CK IDs: T1087 and T1059.003
- Tactics: Discovery, Execution
- Techniques: Account Discovery, Windows Command Shell
- Detection interpretation

This is one of the strongest parts of the case study because it shows that the activity was not only logged, but enriched with:

rule context,
threat classification,
and MITRE ATT&CK mapping.

That makes the event much more useful for SOC triage than raw Windows logs alone.

Case Study Findings

Based on the screenshots and event evidence, the lab demonstrates the following:

1. Wazuh + Sysmon successfully captured Windows endpoint telemetry

The environment was correctly instrumented, and Sysmon logs were available both locally and inside Wazuh.

2. Repeated failed logon attempts were detected and correlated

The net use activity generated Windows Security Event ID 4625 logs, and Wazuh surfaced a Multiple Windows Logon Failures detection tied to the fake account.

3. Registry Run Key persistence was clearly visible

The reg add action created a Run Key named MaliciousUpdate, and Wazuh captured:

- the modifying process (reg.exe)
- the exact target registry path
- the written value (calc.exe)
4. Suspicious command / process activity was exposed with useful context

The execution chain involving cmd.exe / calc.exe generated Sysmon process creation events and corresponding Wazuh detections.

5. Wazuh added investigation value beyond raw logging

The platform enriched the raw telemetry with:

- rule descriptions
- alert severity
- grouped detections
- MITRE ATT&CK mappings
- MITRE ATT&CK Mapping
- Activity	Evidence in Case Study	MITRE ATT&CK
- Multiple failed logon attempts	Windows Event ID 4625 / Multiple Windows Logon Failures	T1110 – Brute Force
- Suspicious command shell execution	Wazuh rule: Suspicious Windows cmd shell execution	T1059.003 – Windows Command Shell
- Registry Run Key persistence	Run Key modification using reg.exe	T1547.001 – Registry Run Keys / Startup Folder
- Discovery-related shell behavior	Wazuh rule metadata on suspicious cmd event	T1087 – Account Discovery
## Skills Demonstrated

This case study demonstrates hands-on skills relevant to a SOC / Blue Team role:

- Wazuh agent deployment and verification
- Sysmon-based Windows telemetry collection
- Windows event investigation
- Threat hunting in Wazuh
- Failed logon analysis
- Registry persistence detection
- Process creation analysis
- Event-level triage using Wazuh document details
- MITRE ATT&CK mapping of endpoint activity
- Building evidence-backed SOC case studies from raw detections
## Conclusion

This project shows how a Windows endpoint can be monitored with Wazuh + Sysmon to detect and investigate suspicious activity such as failed logon attempts, Run Key persistence, and suspicious command execution.

The screenshots demonstrate not just raw activity generation, but the full SOC workflow:

- generate endpoint activity,
- validate telemetry collection,
- investigate detections in the SIEM,
- inspect event details,
- map the findings to attacker techniques.

As a result, the project serves as a practical SOC detection case study rather than a simple setup lab.
