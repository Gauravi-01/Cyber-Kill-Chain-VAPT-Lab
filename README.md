# ⚔️ End-to-End Cyber Kill Chain Emulation & Lateral Movement Lab

## 📌 Project Overview
This project demonstrates a multi-stage enterprise network compromise structured around the **7-Stage Cyber Kill Chain** framework. Using an attacker-defender setup, I successfully breached a public-facing Linux server perimeter, leveraged local broadcast traffic to harvest domain credentials, decrypted authentication hashes offline, and executed lateral movement to fully compromise an isolated internal Windows workstation.

---

## 🛠️ Lab Architecture & Tech Stack
* **Attacker Machine:** Kali Linux (`192.168.0.108`)
* **Perimeter Server Target:** Metasploitable 2 Linux Server (`192.168.0.112`)
* **Internal Workstation Target:** Windows VM (`192.168.0.113`)
* **Core Tooling:** `Nmap`, `Metasploit Framework`, `Responder`, `Hashcat`, `smbclient`

---

## 💥 Detailed Technical Walkthrough (The Kill Chain)

### 🗺️ Phase 1 & 2: Reconnaissance & Weaponization
* **Action:** Conducted full network service discovery using `nmap -sV -sC -A 192.168.0.112`.
* **Discovery:** Identified a vulnerable public-facing daemon running `vsftpd 2.3.4` on Port 21. 
* **Weaponization:** Staged a matching backdoor command execution payload using the Metasploit Framework (`exploit/unix/ftp/vsftpd_234_backdoor`).

### 📦 Phase 3 & 4: Delivery & Exploitation (Perimeter Breach)
* **Action:** Launched the exploit module against the perimeter server target.
* **Impact:** Exploit triggered successfully, completely bypassing standard authentication. Verified full administrative `root` access on the Linux host (`whoami` -> `root`).

### ⚓ Phase 5 & 6: Credential Theft & Command & Control (C2)
* **Action:** Deployed `Responder` on the attacker machine to passively listen for Link-Local Multicast Name Resolution (LLMNR) and NetBIOS name service queries on the internal subnet.
* **The Flaw:** Simulated a user typographic error on the internal Windows workstation by typing an unroutable network path (`\\serverb`). 
* **The Capture:** Responder poisoned the broadcast response, forcing the Windows VM to negotiate authentication and leaking a `NetNTLMv2` handshake challenge hash.
* **Offline Decryption:** Cleaned the raw token syntax structure and fed it into `Hashcat` using dedicated cryptographic processing arrays (`-m 5600`) against the `rockyou.txt` dictionary wordlist.
* **Result:** Achieved 100% dictionary cracking success instantly, recovering the plaintext user account credentials (`abcd:1234`).

#### 📷 Proof of Concept: Successful Hashcat Recovery
![](./screenshots/hashcat_cracked.png)

---

### 🎯 Phase 7: Lateral Movement & Actions on Objectives (Windows Compromise)
* **The Obstacle:** Initial standard remote code execution (RCE) vectors like EternalBlue failed due to up-to-date operating system patch levels. Additionally, default Windows policies strip local administrative privileges during remote network loop connections.
* **Defensive Bypass:** Conducted a post-exploitation system configuration change by modifying the Windows registry key `LocalAccountTokenFilterPolicy` to `1`. This allowed local accounts to retain full elevated security tokens during network access requests.
* **The Pivot:** Used `smbclient` over Port 445 on Kali Linux to pass the cracked credentials and map against the hidden administrative disk root share:
  ```bash
  smbclient //192.168.0.113/C\$ -U "abcd"%1234
  ```
* **Final Impact:** The terminal successfully transitioned to an interactive `smb: \>` command shell. I achieved unauthorized file system indexing, arbitrary read rights over user directories, and complete corporate data exfiltration validation.

#### 📷 Proof of Concept: Interactive SMB File Share Access Granted
![](./screenshots/smb_success.png)

---

## 🛡️ Strategic Enterprise Remediation Roadmap
To break this specific attack chain in a production environment, an organization must deploy the following mitigation controls:
1. **Break Phase 4 (Exploitation):** Establish aggressive patch cycles for all public-facing services and close legacy/unused communication ports (e.g., FTP Port 21) at the network firewall perimeter.
2. **Break Phase 5 (Credential Theft):** Completely disable LLMNR and NBT-NS protocols via Windows Group Policy Objects (GPO) to prevent local network poisoning attacks.
3. **Break Phase 7 (Lateral Movement):** Implement strict network segmentation separating public web subnets from internal employee workstation subnets, and enforce strong password complexity rules to negate dictionary cracking attempts.
