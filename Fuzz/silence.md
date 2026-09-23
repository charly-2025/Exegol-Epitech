--- 

### Challenge propertie(s) :
- ##### Difficulty : medium
- ##### Techniques : scan, enumeration, website analysis
- ##### tools : nmap, ffuf, gpg, libreoffice calc


### Challenge steps : 

#### Part 1 : user.txt

1. I did a nmap scan , to see which ports are opened, 80 and 22 are opened. 

2. I visited the http website, and tried to find some relevant informations on the website. but nothing relevant found.

3. I enumerated directories with ffuf. And found a special directory named "hidden". When I visited the directory it looks like assets one. A zip file is here , so I downloaded it to unzip it.
![](images_Fuzz/ffuf_silence.png)

4.  I unzip the file and gave me two new file. 
```
[Sep 21, 2026 - 16:51:17 (CEST)] exegol-epitech 10.10.0.8 # unzip ../../root/Downloads/stats.zip 

Archive:  ../../root/Downloads/stats.zip
  inflating: ClimbersStats.xlsx.gpg  
  inflating: .hidden-key            
```

5. I knew that a xlsx is a type of file that can be opened in a calc etc.
```
	libreoffice --calc ClimbersStats.xlsx.gpg
```
But when I open libre office calc with the file , the recovery fail and I can't open it. So I tried to find something about gpg extension.
And this website helped me : https://www.ezeelogin.com/kb/article/import-a-gpg-private-key-and-encrypt-decrypt-messages-using-gpg-807.html

I knew with `ls -a` that I also have a ".hiddenkey" .

So thanks to the website, I tried to decrypt the gpg file with the hidden key.

```
~ gpg --import .hidden-key                  
gpg: key CA39F36CD0B69BF0: "Adam Ondra <adam@climbing.thm>" not changed
gpg: key CA39F36CD0B69BF0: secret key imported
gpg: Total number processed: 1
gpg:              unchanged: 1
gpg:       secret keys read: 1
gpg:  secret keys unchanged: 1

~ gpg --output stat.xlsx --decrypt ClimbersStats.xlsx.gpg

gpg: encrypted with 3072-bit RSA key, ID 78390708D04F7F4E, created 2021-12-07
      "Adam Ondra <adam@climbing.thm>"
gpg: Signature made Tue 07 Dec 2021 04:46:54 PM CET
gpg:                using RSA key 8D50652C1742742DD9E61321CA39F36CD0B69BF0
gpg: Good signature from "Adam Ondra <adam@climbing.thm>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 8D50 652C 1742 742D D9E6  1321 CA39 F36C D0B6 9BF0

~ libreoffice --calc stat.xlsx

```

It shows me a table with username, password etc. 

6. So I tried to connect to ssh testing with those credentials. 

```
ssh janja@10.10.0.65
janja@10.10.0.65's password: 
Linux climbing 6.8.0-136-generic #136-Ubuntu SMP PREEMPT_DYNAMIC Wed Jul  1 21:53:05 UTC 2026 x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
janja@climbing:~$ ls
user.txt
janja@climbing:~$ cat user.txt

```

One of those creds worked and I could connect to "janja" and get the flag in "user.txt".