<div align="center">

# 🎯 HTB: Cohort, Walkthrough by Amaira Safwen
(Born as root)


![Status](https://img.shields.io/badge/status-in_progress-yellow)
![Difficulty](https://img.shields.io/badge/labeled-unknown-lightgrey)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Vibe](https://img.shields.io/badge/vibe-corporate_dystopia-purple)

</div>

---

## ☕ A Quick Word Before We Start

Every HTB box promises a story. Cohort promises a story about a company that does "analytics," which is corporate speak for "we have your data and we're not telling you why." The box opens with SSH, HTTP, HTTPS, and a filtered port sitting at 50001 like a bouncer who's not sure if you're on the list yet. It's a wildcard SSL certificate, an nginx redirect, and a `.htb` hostname that implies subdomains exist but refuses to introduce them. This is the part of the box where the machine quietly judges you for not running `ffuf` yet. So here we are, coffee number one, feroxbuster already spinning, and a genuine suspicion that "cohort" is just a fancy word for "group of people whose passwords we're about to steal."

---

## 📋 Machine Info

| Field          | Value                                             |
| -------------- | ------------------------------------------------- |
| **Target IP**  | `10.129.98.178`                                    |
| **Hostnames**  | `cohort.htb`, wildcard `*.cohort.htb`              |
| **OS**         | Linux (Ubuntu)                                     |
| **Difficulty** | [TBD]                                              |
| **Status**     | ⏳ Recon in progress                               |

---

## 📑 Table of Contents

* [1. Enumeration](#1-enumeration)
* [2. Web Recon: The API That Says Hello](#2-web-recon-the-api-that-says-hello)
* [3. Foothold](#3-foothold)
* [4. User Flag](#4-user-flag)
* [5. Privilege Escalation](#5-privilege-escalation)
* [6. Root Flag](#6-root-flag)
* [7. Full Credential Table](#7-full-credential-table)
* [8. Attack Chain Diagram](#8-attack-chain-diagram)
* [9. Lessons Learned](#9-lessons-learned)
* [10. Tools Used](#10-tools-used)

---

# 1. Enumeration

## 🔎 Port Scan

```text
22     SSH     (OpenSSH 9.6p1 Ubuntu)
80     HTTP    (nginx 1.24.0 → redirects to HTTPS)
443    HTTPS   (nginx 1.24.0, cert for cohort.htb + *.cohort.htb)
50001  filtered unknown
