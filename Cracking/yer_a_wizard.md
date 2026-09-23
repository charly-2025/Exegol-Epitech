---

### Challenge propertie(s) :
- ##### Difficulty : easy
- ##### Techniques : scan, enumeration, FTP anonymous access, hidden files/directories, hash identification, dictionary cracking, hex decoding, directory permission abuse, sudoers misconfiguration, GTFOBins privilege escalation.
- ##### tools : nmap, ftp, name-that-hash, john, rockyou.txt, cyberchef, ssh, aspell.

### Challenge steps :

#### Part 1 : user.txt

1. A basic nmap scan on the target IP.

![[images_cracking/yaw1.png]]

2. I try to connect to FTP with a real account, but it asks for a name and password I don't have. I discover that anonymous login is possible on a lot of misconfigured FTP servers.

![[images_cracking/yaw2.png]]

3. Once connected, `ls -la` to see everything, including hidden files. I find a `.hidden`.

![[images_cracking/yaw3.png]]

3. I try to connect via SSH with several variants of the name (`hagrid`, `Hagrid`, `rubeus`, `Rubeus`) and this password. Nothing works.
	-> `ssh rubeus@10.10.0.65` -> `Permission denied`
	-> `ssh hagrid@10.10.0.65` with `Il0veTheMalefoys` -> `Permission denied`

	Reading the file more carefully, the tone is clearly sarcastic ("trust me, I never lie"). This password is a deliberate decoy, not the real info.

4. I go back to the FTP as anonymous and, looking closer at the listing, I notice an entry I missed the first time: a `...` directory. I move into it.

![[images_cracking/yaw5.png]]

5. `get .reallyHidden` then `cat .reallyHidden` gives a second password, this time the real one.

![[images_cracking/yaw6.png]]

6. With this password and a variant of Hagrid's name, the SSH connection finally succeeds. `user.txt` is directly readable in the home directory (encoded in base64).

![[images_cracking/yaw7.png]]

#### Part 2 : root.txt

1. A `riddle.txt` file sits in hagrid's home, but it's unreadable.

![[images_cracking/yaw8.png]]

	The file does belong to hagrid, but its permissions are set to 0000. However, the permission to access a file and the right to change its permissions are two different things: even at 000, the owner always keeps the right to `chmod` it.
	-> `chmod 600 riddle.txt`

![[images_cracking/yaw9.png]]

2. The file contains a message signed by Professor Dumbledore, followed by 16 hashes. Identifying the type with `name-that-hash`: 128 hex characters, so SHA-512.
	-> `john --format=raw-sha512 --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt`
	-> `john --show hashes.txt`

	14 out of 16 lines crack without any issue. **But none of these passwords end up being useful afterwards**: they're decoys, only there to make it look like a classic list to crack one by one. The real info isn't in them.

3. One line resists the dictionary. Same length as the others (128 characters), but it's not a real SHA-512, it's plain text encoded in hexadecimal. Decoded with CyberChef (recipe "From Hex").
	Output :
	-> `In fact, this was simple, true, the password is ByMerlinBeard!`

	The last line of the file is simply the SHA-512 of this plaintext password, an integrity check, not an obstacle by itself.

4. Connecting as dumbledore with this password.
	-> `ssh dumbledore@10.10.0.65`

	Natural reflex once connected as the "headmaster of the school": I test `sudo -l`. **Nothing.**
	-> `sudo -l` -> `Sorry, user dumbledore may not run sudo on hogwarts.`

5. `id` confirms dumbledore is part of an extra group, `headmaster`. So I look for everything this group gives access to on the system.

![[images_cracking/yaw10.png]]

	Two `.ssh` directories belong to the `headmaster` group. Nothing for harry in this listing, but it points me in a direction: maybe his `.ssh` also has this group, without `find` being able to see it.

6. I run `ls -la` on harry's home to check its permissions.

![[images_cracking/yaw11.png]]

	`x` without `r` for "group" and "others" means I can't list the directory's content, but I can still traverse it if I know the exact name of something inside. That's why `find` didn't return anything for harry: it needs to list the parent directory to find what's in it, which it's denied. But I can still try to access `.ssh` directly without going through a listing.

![[images_cracking/yaw12.png]]

	It works: this subdirectory has its own group owner, `headmaster`, with `r-x` rights for that group, independent of the parent directory's permissions. Since dumbledore is part of this group, he can read this specific content even without ever being able to list `/home/harry` entirely.

7. Retrieving harry's SSH private key and connecting.
	-> `cat harry/.ssh/id_ed25519`
	-> `chmod 600 harry_id_ed25519`
	-> `ssh -i harry_id_ed25519 harry@10.10.0.65`

8. Once connected as harry, I check his sudo rights.
	-> `sudo -l`
	Output :
	-> `(root) NOPASSWD: /usr/bin/aspell`

9. The exploitable function of aspell is officially documented on GTFOBins (category "File read"): https://gtfobins.org/gtfobins/aspell/
	-> `sudo aspell -c "$LFILE"`

	Validation test on a normal file:
	-> `sudo aspell -c /etc/hostname`

	On `/root/root.txt`, the interactive interface doesn't open (file too short, no "misspelling" detectable by the dictionary). Testing a variant of the same flaw (`--conf` instead of `-c`, which tries to parse the file as a config), the content leaks directly in the error message.
	-> `sudo aspell --conf /root/root.txt`
	Output :
	-> `Error: /root/root.txt:1: The key "53565a4a..." is unknown.`

10. The leaked content is encoded in several layers (hex, then base64, then base32). Decoded with CyberChef.
	Root flag obtained.

![[yaw13.png]]