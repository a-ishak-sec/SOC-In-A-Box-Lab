# SOC-in-a-Box: SIEM Setup and Threat Simulation Lab

## Project Overview
This repository documents the architecture and configuration of a local Security Operations Center (SOC) environment. The objective of this lab is to establish a functional SIEM, generate detailed endpoint telemetry, and validate threat detection capabilities through simulated cyber attacks.

**Tools & Technologies Used:**
1. VMware Workstation Pro
2. Ubuntu Server 24.04.4 LTS (Splunk Enterprise Server)
3. Windows 10 (Victim Machine)
4. Splunk Universal Forwarder
5. Sysmon (SwiftOnSecurity Configuration)
6. Atomic Red Team

---

## Phase 1: Environment Setup & Operational Security
To ensure a secure and controlled lab environment, all software binaries were verified before installation to mitigate the risk of introducing unknown malware. 

### 1. Verification of Host Hypervisor
The VMware Workstation Pro installer was downloaded from the official Broadcom portal. 
1. Verified the digital signature of the executable belonged to "Broadcom Inc."
2. Utilized PowerShell's `Get-FileHash` command to confirm the SHA hash matched the official documentation (`10fe3a36f525d88aa133118ab3b5a16b18da88d4aa11b14d74e4164b3fb94ba9`).

![Broadcom Digital Signature](images/Confirming%20VMware%20Installer%20Digital%20Sign...)
![Broadcom Official Hash](images/Broadcom%20File%20Hash%20Official%20VM.png)
![Calculated Installer Hash](images/Confirming%20File%20Hash%20VM.png)

### 2. Operating System Verification
Clean installations of Ubuntu and Windows were prioritized over reusing old local ISOs. 
1. Verified the Windows .exe installer digital signatures.
2. Verified the Ubuntu ISO hashes against the official Canonical documentation.

![Windows Installer Signature](images/Confirming%20Windows%20Installer%20Digital%20Sig...)
![Ubuntu Official Hash](images/Ubuntu%20ISO%20Hash%20Official.png)
![Calculated Ubuntu Hash](images/Confirming%20Ubuntu%20Iso%20Hash.png)

### 3. Virtual Machine Provisioning
Allocated specific hardware resources to balance host performance and lab requirements.
1. **Windows 10 VM:** 60 GB Disk, 4 GB RAM, 2 Cores. Installed VMware Tools for environment management.
2. **Ubuntu VM:** 40 GB Disk, 4 GB RAM, 2 Cores. Executed `sudo apt update && sudo apt upgrade -y` for baseline security patches. 
3. Confirmed network connectivity by successfully pinging the Ubuntu Server (`192.168.20.128`) from the Windows host.

![Windows 10 System Info](images/Windows%2010%20VM%20System%20Info.png)
![Ubuntu System Info](images/Ubuntu%20VM%20System%20Info.png)
![Windows Connection Confirmation](images/Windows%2010%20VM%20Connection%20Confirmed.p...)

---

## Phase 2: SIEM Configuration & Telemetry Routing
Established the central logging infrastructure and configured the victim machine to forward event data.

### 1. Splunk Enterprise Deployment
1. Downloaded Splunk to the Ubuntu server via `wget` and installed using `sudo dpkg -i`.
2. Successfully started the `splunkd` service and accessed the web GUI via `http://192.168.20.128:8000`.
3. Configured Splunk to actively listen for incoming data on Port 9997.

![Splunk Running Confirmed](images/Splunk%20running%20confirmed.png)
![Receiving Port Configuration](images/Receiving%20on%20port%209997.png)

### 2. Universal Forwarder Installation
1. Installed Splunk Universal Forwarder (v10.2.2) on the Windows 10 VM.
2. Pointed the forwarder to the Ubuntu IP (`192.168.20.128`) on Port 9997.
3. Verified the Splunk Forwarder service was active in Windows `services.msc`.

![Splunk Forwarder Running](images/SplunkForwarderRunning.png)
![Splunk IP Link Established](images/Splunk%20Working%20On%20IP%20Link.png)

---

## Phase 3: Advanced Telemetry Configuration
Standard Windows event logs often lack the granularity needed for deep threat hunting. Sysmon was integrated to capture specific execution paths and process histories.

### 1. Sysmon Implementation
1. Downloaded Sysinternals Sysmon to the Windows VM.
2. Applied the SwiftOnSecurity `sysmonconfig-export.xml` configuration to filter out benign noise and capture high-fidelity security events.
3. Executed the installation via elevated Command Prompt: `sysmon64.exe -i sysmonconfig-export.xml -accepteula`.

### 2. Data Normalization
1. Installed the Splunk Add-on for Microsoft Windows on the SIEM.
2. This allowed Splunk to automatically parse Sysmon's raw XML logs into readable, categorized fields like `User`, `CommandLine`, and `ParentImage`.

---

## Phase 4: Threat Simulation & Detection Validation
Conducted three separate attack simulations to verify that the SIEM was correctly ingesting, parsing, and displaying malicious activity.

### Simulation A: Brute Force Authentication
1. **The Attack:** Manually triggered 10 consecutive failed login attempts on the Windows VM to simulate a brute force credential attack.
2. **The Detection:** Queried Splunk for `index=* sourcetype="WinEventLog:Security"`. Identified a massive spike in activity. Investigated individual logs to confirm the presence of `Event ID 4625` (An account failed to log on).

![SIEM Logon Surge](images/Seim%20registering%20surge%20of%20logon%20attempts...)
![Event ID 4625 Detected](images/Seim%20Detecting%20Failed%20Login%20Code%204625.p...)
![Login Fail Details](images/Deeper%20look%20at%20login%20fail%20reason%20device%20e...)

### Simulation B: Suspicious Command Line Discovery
1. **The Attack:** Simulated basic adversary reconnaissance by executing `whoami`, `ipconfig /all`, and `net user` in the Windows command prompt.
2. **The Detection:** Used the search `index="main" source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" | stats count by CommandLine`. Successfully located exactly 5 instances of `ipconfig /all`, 5 of `net user`, and 4 of `whoami`.
3. **Deep Dive:** Used a targeted query referencing `ParentImage` to prove that `C:\Windows\System32\cmd.exe` was the root source of the execution, confirming Sysmon's tracking capabilities.

### Simulation C: Persistence via Atomic Red Team
1. **The Attack:** Downloaded and installed the Invoke-AtomicRedTeam execution engine via PowerShell. Triggered a test simulating an attacker establishing persistence via Registry Run Keys: `Invoke-AtomicTest T1547.001 -TestNumbers 1 -PathToAtomicsFolder "C:\AtomicRedTeam\atomics\atomics" -Force`.
2. **The Detection:** Searched Splunk for PowerShell parent images (`index=* (Image="*powershell.exe" OR ParentImage="*powershell.exe") | table _time, Computer, User, CommandLine, ParentCommandLine | sort - _time`). Successfully identified the exact timestamp and the `reg.exe` commands executing the persistence attack.

![Atomic Red Team Installed](images/Invoke%20red%20team%20is%20active%20and%20installed.png)
![Registry Attack Executing](images/Atomic%20Red%20Team%20registry%20attack%20running.png)
![Attack Run Success](images/Atomic%20Red%20Team%20Reg%20Key%20Run%20Success.png)
![Attack Logged in SIEM](images/Atomic%20Red%20Team%20Reg%20Key%20Logged.png)

---

## Challenges Faced

1. **Splunk Root Execution Deprecation:** During the Ubuntu Splunk installation, running the start command threw a warning that root execution was deprecated. After researching the warning, I purposefully bypassed it using `--run-as-root` to keep the lab environment strictly focused on ingestion rather than complex Linux permission management.
2. **SIEM Resource Overload:** Initially, the Splunk dashboard showed no logs and displayed a yellow warning sign indicating output degradation. The 4 GB RAM allocation on the VM was struggling to process the realtime "All Time" query of the Windows Security logs. Changing the timeframe scope to "Realtime: 30 Minute Window" immediately resolved the latency and allowed the data to populate.
3. **EDR/Antivirus Interference:** When unzipping the Atomic Red Team library, Windows Defender triggered severe alerts (flagging `wacatac.b!ml` and `malgent!MSR`) and actively deleted the simulation scripts. To allow the simulation to proceed, I configured a specific exclusion path in Defender using PowerShell (`Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"`), effectively bypassing the host protection.
