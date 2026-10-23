# Exegol: pentest write-ups

> Built at **Epitech Rennes** · B-SPE-321 · Cybersecurity specialization · September 2026

## Overview

A series of offensive security challenges solved from **Exegol**, a ready-to-use
Docker environment for penetration testing. Each target machine is attacked
from scratch: reconnaissance, initial access, then privilege escalation, until
both flags (`user.txt` and `root.txt`) are captured.

Every write-up is written like a short pentest report: challenge properties
(difficulty, techniques, tools) followed by reproducible steps with commands
and screenshots.

## Write-ups

| Category | Challenge | Difficulty | Main techniques |
| --- | --- | --- | --- |
| Fuzz | [h4ck3rz](Fuzz/h4ck3rz.md) | Easy | Scanning, enumeration, website analysis |
| Fuzz | [toss_a_coin](Fuzz/toss_a_coin.md) | Easy | Scanning, enumeration, website analysis |
| Fuzz | [silence](Fuzz/silence.md) | Medium | Directory fuzzing, hidden archive, GPG decryption |
| Injection | [batman's_secret](Injection/batman's_secret.md) | Easy | Anonymous FTP, source code disclosure, OS command injection, sudoers abuse |
| Injection | [fun_with_functional](Injection/fun_with_functional.md) | Easy | Code execution via file upload, reverse shell, weak file permissions |
| Injection | [year_5739](Injection/year_5739.md) | Easy | Server-side template injection to RCE, privilege escalation |
| Injection | [demon_slayer](Injection/demon_slayer.md) | Medium | Web fuzzing, misidentified service, client-side JS analysis, SQL injection |
| Cracking | [yer_a_wizard](Cracking/yer_a_wizard.md) | Easy | Anonymous FTP, hidden files, hash identification, dictionary cracking |
| Cracking | [musa_troglodytarum](Cracking/musa_troglodytarum.md) | Easy | Steganalysis, dictionary brute force, privilege escalation |
| Cracking | [mefferts](Cracking/mefferts.md) | Medium | Steganalysis, dictionary brute force, service authentication |
| Reverse | [jormungandr](Reverse/jormungandr.md) | Medium | Pickle deserialization, PyInstaller decompilation, SSH pivoting, sudoers misconfiguration |

## Methodology

1. **Reconnaissance**: port and service scanning (`nmap`), web content discovery (`ffuf`)
2. **Initial access**: exploiting the weakest entry point (injection, exposed files, weak credentials...)
3. **Privilege escalation**: local enumeration, misconfigurations, vulnerable binaries
4. **Documentation**: every step is reproducible from the write-up alone

## Tech stack

`Exegol` · `nmap` · `ffuf` · `John the Ripper` · `Hashcat` · `gpg` · `netcat` · `Python` · `Linux`

## Disclaimer

All challenges were solved on intentionally vulnerable machines provided for
training. These techniques must only be used on systems you are authorized to
test.

---

More of my work on my [portfolio](https://charly-2025.github.io/myPortfolio/).
