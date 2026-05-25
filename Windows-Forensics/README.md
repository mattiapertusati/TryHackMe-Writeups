# TryHackMe: Investigating Windows — DFIR Write-Up

## 📝 Room Description
A Windows production server has been compromised. As a Digital Forensics and Incident Response (DFIR) analyst, my task is to investigate the volatile and non-volatile artifacts on this Windows machine, uncover the attacker's footprint, map out their timeline, and isolate the tools and infrastructure used during the breach.

* **Target OS:** Windows Server
* **Access Vector:** Remote Desktop Protocol (RDP)
* **Initial Credentials Provided:** `Administrator` : `letmein123!`

---

## 🔍 Forensic Investigation & Write-Up

### Q1: What is the version and year of the Windows machine?
* **Objective:** Establish the target system identity and OS baseline.
* **Methodology:** After establishing the RDP session, I opened an administrative Command Prompt (CMD) and executed the `systeminfo` utility to extract detailed OS configuration data.
* **Command:**
  ```cmd
  systeminfo | findstr /B /C:"OS Name"
  ```
* **Artifact/Result:** **Windows Server 2016**

---

### Q2: Which user logged in last?
* **Objective:** Determine the most recent active interactive session before or during the incident.
* **Methodology:** I used the `query user` command to inspect current active sessions, which returned the `Administrator` account. To cross-reference this and ensure the attacker had not hijacked a different session, I checked the Windows Security Log via Event Viewer for Event ID `4624` (Successful Logon).
* **Command:**
  ```cmd
  query user
  ```
* **Artifact/Result:** **Administrator**

---

### Q3: When did John log onto the system last?
* **Objective:** Audit local user account activity to spot suspicious logons.
* **Methodology:** I listed all local user accounts using `net user` to map out the attack surface. Upon identifying the user `John`, I ran a deep query on his specific profile to extract the exact `Last logon` timestamp.
* **Command:**
  ```cmd
  net user John
  ```
* **Artifact/Result:** **03/02/2019 5:48:32 PM**

---

### Q4: What IP does the system connect to when it first starts?
* **Objective:** Detect unauthorized persistence mechanisms embedded into the system boot sequence.
* **Methodology:** I utilized the Windows Management Instrumentation Command-line (`wmic`) tool to audit startup applications. This bypasses basic GUI hiding techniques.
* **Command:**
  ```cmd
  wmic startup get caption,command
  ```
* **Analysis:** The output revealed a malicious entry disguised as a system service named `UpdateSvc` (a classic *Masquerading* technique). Instead of a legitimate Windows update binary, it was executing a hidden binary pointing to an external command-and-control listener.
* **Artifact/Result:** **10.34.2.3**

---

### Q5: What two accounts had administrative privileges (other than the Administrator user)?
* **Objective:** Identify local privilege escalation and backdoor accounts.
* **Methodology:** I queried the local administrators group database to find accounts that had been granted elevated execution rights.
* **Command:**
  ```cmd
  net localgroup administrators
  ```
* **Analysis:** The output revealed that the attacker abused the built-in, typically disabled `Guest` account by adding it to the high-privilege group, alongside a newly created or compromised user named `Jenny`.
* **Artifact/Result:** **Guest, Jenny** (Alphabetical order)

---

### Q6: What is the name of the scheduled task that is malicious?
* **Objective:** Identify time-based persistence mechanisms.
* **Methodology:** I opened **Task Scheduler** (`taskschd.msc`) and focused on the *Task Scheduler Library*. Matching the timeline of the compromise (immediately following John's logon on `03/02/2019`), I isolated three anomalous scheduled tasks:
  1. `Check logged in` $\rightarrow$ Executing a spoofed binary in a faked directory: `"C:\Program Files (x86)\Internal Explorer\iexplore.exe"` *(Typo-squatting Internet Explorer)*.
  2. `Clean file system` $\rightarrow$ Executing a local PowerShell-based Netcat script: `"C:\TMP\nc.ps1 -l 1348"`.
  3. `flashupdate22` $\rightarrow$ Executing an obfuscated PowerShell one-liner: `powershell.exe -WindowStyle Hidden -nop -c ""`.
* **Artifact/Result:** **Clean file system** *(This task explicitly opens an active backdoor via Netcat on port 1348)*.

---

### Q7: At what date did the compromise take place?
* **Objective:** Build an accurate incident timeline.
* **Methodology:** Correlating the creation timestamps of the three malicious scheduled tasks, the `Last logon` date of the user John, and file modification artifacts within `C:\TMP\`, the operational window of the attacker was confirmed.
* **Artifact/Result:** **03/02/2019**

---

### Q8: During the compromise, at what time did Windows first assign special privileges to a new logon?
* **Objective:** Find the precise second the attacker gained elevated access.
* **Methodology:** I audited the Windows Security Event Log for **Event ID 4672** (*Special privileges assigned to new logon*). I cross-referenced this log around the known attack date window to pinpoint the exact time block.
* **Command (CLI Alternative):**
  ```cmd
  wevtutil qe Security /q:"*[System[(EventID=4672)]]" /f:text /rd:true /c:20
  ```
* **Artifact/Result:** **03/02/2019 05:48:32 PM**

---

### Q9: What tool was used to get Windows passwords?
* **Objective:** Identify the post-exploitation credential dumping tooling.
* **Methodology:** I inspected the volatile staging directory `C:\TMP\` (a standard directory often left with weak ACLs where attackers drop payloads). Inside, I discovered remnants of an execution output file and a renamed binary named `mim.exe`.
* **Analysis:** The execution artifacts matched the signature of **Mimikatz**, an open-source credential dumping tool used to extract plaintext passwords and LSASS memory hashes. 
* **Discovered Credential Leak:** The Mimikatz log successfully dumped credentials for the user `Ion` $\rightarrow$ Password: `MySecretP4ass`.
* **Artifact/Result:** **Mimikatz**

---

### Q10: What was the attacker's external command and control server IP?
* **Objective:** Isolate the attacker's C2 infrastructure.
* **Methodology:** Attackers frequently manipulate local name resolution to intercept traffic or hardcode callback destinations. I reviewed the Windows static lookup file located at `C:\Windows\System32\drivers\etc\hosts`.
* **Command:**
  ```cmd
  type C:\Windows\System32\drivers\etc\hosts
  ```
* **Analysis:** The bottom of the file contained an unauthorized record routing a major domain to a foreign, untrusted public IP address.
* **Artifact/Result:** **76.32.97.132**

---

### Q11: What was the extension name of the shell uploaded via the server's website?
* **Objective:** Identify the initial web-facing entry vector.
* **Methodology:** Since the machine functions as a web server, I audited the Internet Information Services (IIS) default web root directory (`C:\inetpub\wwwroot\`) to check for unauthorized file uploads (*Web Shells*).
* **Command:**
  ```cmd
  dir C:\inetpub\wwwroot\
  ```
* **Analysis:** Alongside standard asset files (like `.gif` items), I spotted a server-side executable script written with a **.jsp** (JavaServer Pages) extension, confirming the application layer was leveraged to run remote commands.
* **Artifact/Result:** **.jsp**

---

### Q12: What was the last port the attacker opened?
* **Objective:** Identify firewall alterations designed to allow inbound persistent connections.
* **Methodology:** I reviewed the Windows Advanced Firewall Rules console (`wf.msc`) for newly introduced Inbound Rules lacking official Microsoft Group signatures.
* **Analysis:** I identified a custom rogue rule named `"Allow outside connections for development"`. Inspecting its properties under the *Protocols and Ports* tab revealed an open port listener.
* **Artifact/Result:** **1337**

---

### Q13: Check for DNS poisoning — which site has been targeted?
* **Objective:** Determine the objective of the network redirection attack.
* **Methodology:** Returning to the audited `C:\Windows\System32\drivers\etc\hosts` file analyzed in Q10, I inspected the domain mapped to the malicious C2 infrastructure (`76.32.97.132`).
* **Analysis:** The attacker spoofed the local resolution of a critical domain to redirect traffic away from legitimate DNS servers.
* **Artifact/Result:** **google.com**

---

## 🛠️ Key Takeaways & Skills Demonstrated
* **Incident Timeline Reconstruction:** Successfully mapped user logins, file creations, and event log entries to form a coherent chain of events.
* **Persistence & Detection Analysis:** Uncovered hidden scheduled tasks, malicious startup registry entries (`UpdateSvc`), and unauthorized firewall openings.
* **Artifact Analysis:** Identified critical Indicators of Compromise (IoCs) including web shells (`.jsp`), credential dumpers (`Mimikatz`), and local name resolution manipulation (*DNS Poisoning via Hosts file*).

<img width="1365" height="836" alt="Screenshot 2026-05-25 211103" src="https://github.com/user-attachments/assets/99447e52-a082-4efa-9d1a-901ed40a67a0" />

