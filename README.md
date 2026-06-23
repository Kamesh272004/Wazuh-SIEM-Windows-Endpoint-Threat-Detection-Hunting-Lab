# Wazuh SIEM Windows Endpoint Threat Detection & Hunting Lab

## Project Overview
This project is a **Windows endpoint threat detection and hunting lab** built using **Wazuh** and **Sysmon**.  
The goal of this lab was to simulate suspicious activities on a **Windows 10 endpoint**, forward the resulting telemetry into **Wazuh**, and validate whether the platform could detect, classify, and expose the activity through its **Threat Hunting** interface.

Rather than being just a setup lab, this project was executed like a **mini SOC case study**:

- suspicious endpoint activity was manually generated on the Windows host,
- Wazuh detections were reviewed,
- event details were investigated,
- and the alerts were mapped to attacker behavior and MITRE ATT&CK techniques.

The lab focuses on **four major activity clusters**:

1. **Endpoint telemetry verification using Wazuh + Sysmon**
2. **Multiple failed Windows logon attempts**
3. **Registry Run Key persistence using `reg.exe`**
4. **Suspicious process / command shell execution involving `calc.exe`**

---

## Objective

The objective of this lab was to answer the following SOC-style question:

> **If suspicious Windows endpoint activity is executed on a monitored host, can Wazuh + Sysmon capture enough telemetry to detect it, classify it, and support investigation?**

To answer this, the endpoint was instrumented with Wazuh and Sysmon, and several suspicious actions were performed and investigated through Wazuh.

---

## Lab Environment

### Monitoring Stack
- **SIEM / XDR Platform:** Wazuh
- **Endpoint:** Windows 10
- **Telemetry Source:** Sysmon + Windows Security Event Logs
- **Attack / Support System:** Kali Linux
- **Shells Used:** Command Prompt and PowerShell

---

## Lab Workflow

This lab was executed in the following sequence:

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

Before generating suspicious activity, the first step was to verify that the endpoint was properly instrumented.

## What was verified
- Wazuh service running on Windows
- Sysmon service running on Windows
- Recent Sysmon events visible from PowerShell
- Wazuh agent configuration present and pointing to the manager

## 1) Wazuh Agent + Sysmon Verification
![Wazuh Agent and Sysmon Verification](wazuh%20agent%20and%20sysmon%20verification.jpg)

## Findings
The screenshot confirms that:

- the **Wazuh service** started successfully,
- **Sysmon** was installed and actively running,
- Sysmon event data could be retrieved from  
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

## Scenario A – PowerShell Shortcut / LNK Activity

A PowerShell-based shortcut creation sequence was executed to generate suspicious command activity and produce endpoint telemetry associated with user execution / suspicious script behavior.

## 2) PowerShell Shortcut / LNK Creation
> Add your PowerShell screenshot here later if needed.

## What the activity involved
PowerShell commands were used to create a Windows shortcut (`.lnk`) object via `WScript.Shell`.

The shortcut was configured to launch:
- **Target path:** `cmd.exe`
- **Arguments:** `/c calc.exe`

An initial one-line attempt produced an error, after which the shortcut creation steps were executed line by line and saved successfully.

## Why this matters
From a SOC perspective, PowerShell-based shortcut creation is relevant because:

- it shows scripted user-execution behavior,
- it can be abused for phishing or persistence,
- and it produces endpoint telemetry useful for detection engineering.

This activity also complements later detections involving **cmd.exe** and **calc.exe**.

---

## Scenario B – Failed Windows Logon Attempts

The next activity was to generate failed authentication attempts using a fake user account.

### Command Used

```cmd
net use \\localhost /u:FakeHackerAccount Password123
```

## 3) Failed Logon Attempts via Command Prompt
![Failed Logon Attempts via Command Prompt](Failed%20log%20on.jpg)

## What the screenshot shows
The command prompt output shows repeated failed attempts using the username:

`FakeHackerAccount`

The Windows system returns:

```cmd
System error 1326 has occurred.
The user name or password is incorrect.
```

## Why this matters
This activity is useful for validating whether:

- Windows Security logon failure events are generated,
- Wazuh ingests those events,
- and repeated failures are surfaced as a suspicious authentication pattern.

---

## Scenario C – Registry Run Key Persistence

A persistence mechanism was created by adding a malicious autorun registry entry under the current user Run key.

### Command Used

```cmd
reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run" /v "MaliciousUpdate" /t REG_SZ /d "C:\Windows\System32\calc.exe" /f
```

## 4) Registry Persistence Creation via reg.exe
![Registry Persistence Creation via reg.exe](Registry%20presistence.jpg)

## What the screenshot shows
The command prompt shows successful creation of a registry value named:

`MaliciousUpdate`

pointing to:

```cmd
C:\Windows\System32\calc.exe
```

under the current user Run path.

## Why this matters
Run Keys are a common Windows persistence mechanism. A SOC analyst should be able to determine:

- which process modified the registry,
- what registry path was touched,
- and what executable was configured to launch on user logon.

---

## Scenario D – Kali Reconnaissance Activity

A Kali Linux host was also used in the lab to perform reconnaissance activity against the Windows system.

## 5) Kali Nmap Scan / Recon Activity
> Add your Kali screenshot here later if needed.

The Kali terminal was used to perform reconnaissance activity against the Windows endpoint as part of the wider lab workflow.

---

# Phase 3 – Wazuh Threat Hunting Review

After generating suspicious activity, the next step was to investigate the endpoint in **Wazuh Threat Hunting** and validate which detections appeared.

## 6) Sysmon Events Overview in Wazuh
![Sysmon Events Overview in Wazuh](alerts%20in%20wazuh.jpg)

## What this screenshot shows
This screenshot provides the overall Wazuh event view for the Windows endpoint and shows that the platform captured multiple relevant detections, including:

- **Suspicious Windows cmd shell execution**
- **Windows command prompt started by an abnormal process**
- **PowerShell execution**
- **Executable file dropped in a folder commonly used by malware**
- other **Sysmon-based detections**

This confirms that the endpoint telemetry was successfully flowing into Wazuh and that multiple activity types were visible for investigation.

---

# Phase 4 – Failed Logon Investigation

The failed logon scenario was then investigated in detail.

## 7) Multiple Logon Failures Detection
![Multiple Logon Failures Detection](suspicios%20logon%20detected.png)

## What this screenshot shows
Wazuh detected repeated authentication failures and surfaced them as a higher-level alert:

**Multiple Windows Logon Failures**

This indicates that Wazuh did not merely store isolated failed logon events — it recognized the repeated pattern.

---

## 8) Failed Logon Event Details
![Failed Logon Event Details](suspicious%20logon%20investigation.png)

## Key evidence visible in the event details
The event details show:

- target username: **FakeHackerAccount**
- Windows Security **Event ID 4625**
- failed logon metadata such as:
  - logon type
  - status / substatus codes
  - target user fields
  - workstation information

## Detection interpretation
This confirms that the repeated `net use` commands successfully generated Windows Security logon failure events and that Wazuh could ingest and expose them for investigation.

---

# Phase 5 – Registry Persistence Investigation

The registry persistence activity was then validated inside Wazuh.

## 9) Registry Run Key Detection
![Registry Run Key Detection](registry%20presistence.png)

## What this screenshot shows
Wazuh detected the registry modification and raised an alert describing:

**Registry entry to be executed on next logon was modified using command line application reg.exe**

This confirms that the persistence action was not only logged by Sysmon, but also interpreted by Wazuh as a suspicious Run Key modification.

---

## 10) Registry Run Key Event Details
![Registry Run Key Event Details](regitry%20presistence%20investiogation.png)

## Key evidence visible in the event details
The detailed event view shows:

- **Sysmon Event ID 13** (registry value set / registry modification)
- **Image:** `C:\Windows\system32\reg.exe`
- **TargetObject:** path ending in  
  `\Software\Microsoft\Windows\CurrentVersion\Run\MaliciousUpdate`
- **Details:** `C:\Windows\System32\calc.exe`

## Detection interpretation
This proves that the exact persistence action performed from the command line was captured at the telemetry level and surfaced with the key forensic fields needed for investigation:

- the process that made the change,
- the registry object modified,
- and the value written.

---

# Phase 6 – Suspicious Registry Value / Secondary Detection

In addition to the Run Key alert, Wazuh also surfaced another detection tied to the registry value.

## 11) Base64-like Registry Value Detection
![Base64-like Registry Value Detection](privilage%20and%20presistence%20detection.png)

## What this screenshot shows
The Wazuh alert describes:

**Value added to registry key has Base64-like pattern**

Although the registry value in this lab pointed to `calc.exe`, the screenshot demonstrates that Wazuh has content-based registry detection logic capable of flagging suspicious values based on pattern analysis.

## Why this matters
This is useful from a SOC perspective because registry monitoring is not limited to **where** a value was written — it can also include logic about **what the value looks like**.

---

# Phase 7 – Suspicious Process / Command Execution Investigation

The next part of the case study focused on the execution of `calc.exe` and related suspicious command shell activity.

---

## A) Calc Process Detection

## 12) Calc Process Detection
![Calc Process Detection](suspicious%20shell%20command%20.png)

## What this screenshot shows
The Wazuh event view shows a detection associated with:

`data.win.eventdata.image:*calc.exe*`

The rule description visible in the screenshot is:

**Suspicious Windows cmd shell execution**

This indicates that the `calc.exe` execution was visible in the monitored telemetry and linked to suspicious shell behavior.

---

## 13) Calc Process Event Details
![Calc Process Event Details](suspicious%20command%20shell%20investigation.png)

## Key evidence visible in the event details
The detailed Sysmon event includes:

- **Image:** `C:\Windows\System32\calc.exe`
- **Original filename:** `CALC.EXE`
- **Description:** Windows Calculator
- **CommandLine:** `calc.exe`

## Detection interpretation
This confirms that Sysmon process creation logging was working correctly and that the execution of `calc.exe` could be traced in detail through Wazuh.

---

## B) Suspicious CMD Shell Detection

## 14) Suspicious CMD Execution Detection
![Suspicious CMD Execution Detection](account%20discovery.png)

## What this screenshot shows
Wazuh flagged a detection with the rule description:

**Suspicious Windows cmd shell execution**

The event list associates this activity with the monitored Windows endpoint and places it within the same time window as the other suspicious actions.

---

## 15) Suspicious CMD Execution Event Details
![Suspicious CMD Execution Event Details](presistence%20and%20privilage%20investigation.png)

## Key evidence visible in the event details
The detailed event panel shows:

- Sysmon-based telemetry
- Wazuh rule metadata
- Rule description: **Suspicious Windows cmd shell execution**
- MITRE ATT&CK IDs: **T1087** and **T1059.003**
- Tactics: **Discovery**, **Execution**
- Techniques:
  - **Account Discovery**
  - **Windows Command Shell**

## Detection interpretation
This is one of the strongest parts of the case study because it shows that the activity was not only logged, but enriched with:

- rule context,
- threat classification,
- and MITRE ATT&CK mapping.

That makes the event much more useful for SOC triage than raw Windows logs alone.

---

# Case Study Findings

Based on the screenshots and event evidence, the lab demonstrates the following:

## 1) Wazuh + Sysmon successfully captured Windows endpoint telemetry
The environment was correctly instrumented, and Sysmon logs were available both locally and inside Wazuh.

## 2) Repeated failed logon attempts were detected and correlated
The `net use` activity generated **Windows Security Event ID 4625** logs, and Wazuh surfaced a **Multiple Windows Logon Failures** detection tied to the fake account.

## 3) Registry Run Key persistence was clearly visible
The `reg add` action created a Run Key named `MaliciousUpdate`, and Wazuh captured:

- the modifying process (`reg.exe`)
- the exact target registry path
- the written value (`calc.exe`)

## 4) Suspicious command / process activity was exposed with useful context
The execution chain involving `cmd.exe` / `calc.exe` generated Sysmon process creation events and corresponding Wazuh detections.

## 5) Wazuh added investigation value beyond raw logging
The platform enriched the raw telemetry with:

- rule descriptions
- alert severity
- grouped detections
- MITRE ATT&CK mappings

---

# MITRE ATT&CK Mapping

| Activity Evidence in Case Study | MITRE ATT&CK |
|---|---|
| Multiple failed logon attempts | **T1110 – Brute Force** |
| Suspicious command shell execution | **T1059.003 – Windows Command Shell** |
| Registry Run Key persistence | **T1547.001 – Registry Run Keys / Startup Folder** |
| Discovery-related shell behavior | **T1087 – Account Discovery** |

---

# Skills Demonstrated

This lab demonstrates hands-on skills relevant to a **SOC / Blue Team** role:

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

---

# Conclusion

This project demonstrates how a Windows endpoint can be monitored using **Wazuh + Sysmon** to detect and investigate suspicious activity such as:

- repeated failed logon attempts,
- registry Run Key persistence,
- suspicious command shell execution,
- and abnormal process activity involving `calc.exe`.

The lab goes beyond simply forwarding logs into a SIEM. It shows a full **SOC-style workflow**:

1. **generate suspicious endpoint activity**
2. **verify telemetry collection**
3. **hunt detections inside Wazuh**
4. **inspect event details**
5. **map findings to attacker behavior and MITRE ATT&CK**

From a defensive security perspective, the case study highlights the value of combining **Sysmon telemetry** with **Wazuh detection logic**.  
Sysmon provides detailed endpoint visibility for process creation, registry modification, and command execution, while Wazuh adds investigation value by correlating activity, assigning rule context, surfacing severity, and mapping alerts to adversary techniques.

Overall, this project serves as a practical **Windows endpoint threat detection and hunting lab** that demonstrates:

- endpoint telemetry validation,
- alert triage,
- threat hunting in Wazuh,
- persistence detection,
- failed logon investigation,
- and process-level forensic analysis.

It can be used as a portfolio project to showcase hands-on **SOC Analyst / Blue Team / Detection Engineering** skills in areas such as Windows telemetry analysis, Wazuh threat hunting, and ATT&CK-aligned investigation.
