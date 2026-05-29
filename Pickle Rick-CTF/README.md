# TryHackMe: Pickle Rick Walkthrough

![TryHackMe](https://shields.io)
![Difficulty](https://shields.io)

A comprehensive walkthrough for the **Pickle Rick** CTF room on TryHackMe. This room is a Rick and Morty-themed challenge requiring web enumeration, initial access exploitation, and basic Linux privilege escalation.

## Room Information

- **Target IP:** `10.113.145.234`
- **Objective:** Find 3 ingredients to help Rick turn back into a human.
- **Difficulty:** Easy

---

## Phase 1: Reconnaissance & Enumeration

### 1. Nmap Port Scan
I initiated the assessment with an aggressive Nmap scan to discover open ports and running services.

```bash
sudo nmap -sC -sV -T4 -Pn 10.113.145.234
```

**Results:**
```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey: 
|   3072 0f:6d:bd:d4:38:84:5c:67:cb:88:e1:38:a5:d3:0b:8b (RSA)
|   256 33:b9:fe:ce:9b:53:39:f3:45:5e:11:51:5f:85:92:5d (ECDSA)

|_  256 8b:d6:52:59:51:46:43:2b:0d:bc:db:27:d4:0d:65:9c (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Rick is sup4r cool
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The scan revealed two open ports:
- **Port 22 (SSH)**: Open, running OpenSSH 8.2p1.
- **Port 80 (HTTP)**: Open, running Apache web server.

### 2. Web Enumeration
Visiting the website at `http://10.113.145.234` showed a basic, landing page. Checking the HTML source code revealed a hidden developer comment containing a cleartext username:

- **Username:** `R1ckRul3s`

Next, I used **Gobuster** to fuzz the web directories and look for hidden pages, targeting specific file extensions (`.php`, `.html`, `.txt`).

```bash
gobuster dir -u http://10.113.145.234 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
```

The tool discovered two critical assets:
1. `http://10.113.145.234/login.php` (A login portal)
2. `http://10.113.145.234/robots.txt`

Inspecting `robots.txt` revealed a single plaintext string: `Wubbalubbadubdub`.

---

## Phase 2: Initial Access & Exploitation

Using the gathered information, I successfully authenticated into `login.php`:
- **Username:** `R1ckRul3s`
- **Password:** `Wubbalubbadubdub`

Upon logging in, I was redirected to a Command Panel dashboard that allowed remote execution of Linux commands. 

To bypass the restrictive web interface and get an interactive shell, I executed a **Netcat reverse shell** command through the panel, gaining a connection back to my local machine.

### Ingredient #1
Running basic file listing commands (`ls -la`) in the current web root directory (`/var/www/html`) revealed the first objective:
* **First Ingredient:** `mr. meeseek hair`

---

## Phase 3: Post-Exploitation & Privilege Escalation

### Ingredient #2
With a working shell as the web user (`www-data`), I navigated to the `/home` directory to perform local enumeration. Inside `/home/rick`, I located the second ingredient:
* **Second Ingredient:** `1 jerry tear`

### 1. Privilege Escalation to Root
To find the final ingredient, I checked the current user's privileges by running:

```bash
sudo -l
```

The output indicated that the `www-data` user had blanket permissions to run **any** command as `root` without providing a password (`(ALL : ALL) NOPASSWD: ALL`).

I immediately upgraded to root access by executing:

```bash
sudo su
```

### Ingredient #3
Now acting as the root user, I navigated directly to the `/root` directory. Inside, I found the third and final file:
* **Third Ingredient:** `fleeb juice`

---

## Conclusion
The Pickle Rick room was successfully completed by leveraging basic web enumeration, discovering leaked credentials in the source code/robots.txt, exploiting a Command Injection vulnerability to drop a reverse shell, and taking advantage of a misconfigured `sudoers` file for instant root access.
