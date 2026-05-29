# 🛡️ Cybersecurity Labs & Write-ups Portfolio

Welcome to my cybersecurity portfolio! This repository contains detailed, professional write-ups of machines and challenges I have completed on platforms like TryHackMe and other competitive environments.

My goal is to document my analytical process, demonstrate hands-on skills in Penetration Testing, DFIR (Digital Forensics and Incident Response), and Security Operations (SOC), and showcase how I approach real-world security incidents.

---

## 🗂️ Repository Structure

### 📂 Linux-Machines
Detailed technical write-ups covering Linux exploitation, misconfigurations, and CVE analysis.
*   **[Pickle Rick (TryHackMe)](./Pickle-RIck-CTF/README.md)** — Web infrastructure fuzzing, source code analysis, exploiting a command injection vulnerability to drop a reverse shell, and leveraging a misconfigured sudoers file for instant root privilege escalation.
*   **[GoldenEye (TryHackMe)](./GoldenEye-CTF/README.md)** — Pivoting through internal POP3 mail services, steganographic analysis, Moodle CMS spellchecker exploitation, and local privilege escalation via OverlayFS kernel exploit.
*   **[CVE-2026-42945: Nginx Rift](./CVE-2026-42945-Nginx-Rift/README.md)** — Unauthenticated heap overflow walkthrough and code execution analysis.

### 📂 Windows-Forensics
Deep dives into Windows event logs, registry analysis, persistence mechanisms, and artifact hunting.
*   **[Investigating Windows](./Windows-Forensics/README.md)** — Investigating a compromised Windows Server, uncovering Mimikatz artifacts, web shells, and DNS poisoning.

### 🔐 Cryptography & Custom Tooling
Focus on reverse engineering nested ciphers, obfuscation, and automation.
*   **[Break It (TryHackMe)](./Break-It-CTF/README.md)** — Defeating multi-layered "Matryoshka" cryptography by developing a custom heuristic-based Python decoder.

---

## 🛠️ Skills & Tools Demonstrated

*   **Penetration Testing & Exploitation:** Network mapping (**Nmap**), web directory fuzzing (**Gobuster**), service brute-forcing (**THC-Hydra**), web application command injection, interactive reverse shells, and local kernel exploitation (CVE-2015-1328).
*   **Privilege Escalation:** Auditing **sudoers** configuration (`sudo -l`), Linux capabilities, and abusing SUID binaries.
*   **DFIR:** Timeline reconstruction, Windows Event Log analysis (`wevtutil`), and Task Scheduler auditing.
*   **Network & Email Security:** Analyzing SMTP/POP3 protocols, banner grabbing, and local DNS routing manipulation (`/etc/hosts`).
*   **Malware & Artifact Analysis:** Identifying web shells (`.jsp`), image steganography extraction (**Exiftool**), detecting process masquerading, and typo-squatting.
*   **Cryptography & Automation:** Multi-layered decoding (Base58/85/91, ROT47, Vigenère), heuristic language scoring, and AI-assisted Python tool development.
