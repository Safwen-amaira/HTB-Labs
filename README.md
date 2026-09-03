# HTB-Labs

My journey into cybersecurity through Hack The Box labs, one machine at a time.

Welcome to HTB-Labs  my personal repository for documenting my journey as a cybersecurity apprentice through {"fallbackMarkdown":"Hack The Box","reference":{"alt":"Hack The Box","category":"company","extra_params":{"disambiguation":"cybersecurity training platform"},"name":"Hack The Box","prompt_text":"Hack The Box","status":"done","type":"entity"},"referenceKey":"0","showLoginRequiredCard":false} labs.

This repository contains my walkthroughs, notes, methodologies, commands, discoveries, and lessons learned while working through Hack The Box challenges and machines.

The goal isn't simply to collect flags. It's to build a strong understanding of how penetration testing works, develop better problem-solving habits, and document the learning process along the way.

# 🎯 Goals

Through this journey, I aim to develop practical skills in:

🔎 Reconnaissance and enumeration
🌐 Web application security
🐧 Linux and Windows environments
🔐 Privilege escalation
📡 Network enumeration
🧩 Vulnerability analysis
🛠️ Penetration-testing methodology
🐚 Shells and post-exploitation
📜 Bash and Python scripting
🧠 Problem solving and investigative thinking
📝 Professional security documentation
📂 Repository Structure

The repository will evolve as I progress through different labs.
```bash
HTB-Labs/
│
├── Machines/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
│
├── Challenges/
│   ├── Web/
│   ├── Crypto/
│   ├── Forensics/
│   ├── Reverse-Engineering/
│   └── Pwn/
│
├── Notes/
│   ├── Enumeration/
│   ├── Privilege-Escalation/
│   ├── Web-Security/
│   └── Cheat-Sheets/
│
├── Scripts/
│   ├── Bash/
│   └── Python/
│
└── README.md
```

The exact structure may change as I discover better ways to organize my notes.

# 🧪 My Methodology

For each machine or challenge, I try to follow a structured approach:

 ```bash
Reconnaissance
      ↓
Enumeration
      ↓
Service Analysis
      ↓
Vulnerability Identification
      ↓
Initial Access
      ↓
Privilege Escalation
      ↓
Proof / Flag
      ↓
Documentation
      ↓
Lessons Learned
```


I will document not only what worked, but also dead ends and failed approaches when they provide useful lessons.

# 📝 Walkthrough Format

Each machine walkthrough will generally contain:

1. Reconnaissance
Target information
Port scanning
Service discovery
Technology identification
2. Enumeration
Web enumeration
DNS/subdomain enumeration
SMB/FTP/SSH enumeration
Directory and file discovery
User and service enumeration
3. Initial Access
Vulnerability analysis
Exploitation
Credentials or access obtained
Initial shell
4. Privilege Escalation
Local enumeration
Misconfigurations
Interesting files
SUID / capabilities
Scheduled tasks
Service configuration
Credential discovery
5. Proof

Documenting the successful completion of the lab and relevant flags/proofs.

6. Lessons Learned

The most important section:

What did this lab teach me?

I'll record new techniques, concepts, mistakes, and things I should investigate further.

# 🧰 Tools

Some of the tools I expect to use throughout the journey include:

Nmap
Gobuster
Feroxbuster
ffuf
Burp Suite
Netcat
Nikto
WhatWeb
Subfinder
Amass
DNSx
LinPEAS
WinPEAS
Impacket
BloodHound
John the Ripper
Hashcat
Metasploit
Python
Bash

The purpose isn't to memorize tools, but to understand when, why, and how to use them.


# 📈 Learning Progress

I'll use this repository as a record of my progress.

[ ] Networking fundamentals
[ ] Linux fundamentals
[ ] Windows fundamentals
[ ] Web enumeration
[ ] Network enumeration
[ ] Linux privilege escalation
[ ] Windows privilege escalation
[ ] Active Directory
[ ] Web application security
[ ] Scripting & automation
[ ] Advanced exploitation
[ ] Professional reporting


This checklist will evolve as my knowledge grows.

# ⚠️ Disclaimer

This repository is intended for educational purposes and authorized security testing only.

The techniques and tools documented here should only be used against systems where you have explicit permission to test.

Do not use these techniques against unauthorized targets.

# 🧠 Philosophy

Don't just copy the command. Understand the reason behind it.

Every machine is an opportunity to improve:

How I enumerate
How I think about attack surfaces
How I interpret results
How I troubleshoot
How I research
How I document findings

The objective is not to become someone who can simply follow a walkthrough.

The objective is to become someone who can solve the problem without one.

# 🚀 Journey

This repository represents my progress from apprentice → practitioner.

Some write-ups will be clean.

Some will contain mistakes.

Some machines will take hours.

Some will teach me something in minutes.

That's part of the journey.

One lab. One vulnerability. One lesson at a time.

⭐ If you find something useful here, feel free to explore the repository and learn along with me.
