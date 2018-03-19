---
layout: post
title: Nibbles
categories:
- Hack The Box
---

> **Alice:** There is no lock, but it won't open. It's stuck.
>
> **Cheshire Cat:** Think of it as a Chinese Box or a stubborn lid. A tap in the right spot might do the trick.
>
>[Alice looks thoughtfully at The Book of Bizzare Things and kicks it off a ledge; it lands on the ground several floors below, and grudgingly opens]
>
>**Cheshire Cat:** You call that a tap?! Fortunate I didn't suggest "force" – you might have pulverized it. 
> 
>―-American McGee's Alice 


The second machine took a lot less time than the first. Now with some tools under my belt I ventured to tackle the next box. Together with a friend we decided to tackle Nibbles. As with bashed I will first detail the steps we took to gain the root flag followed by the optimal way of doing the same.

Browsing to the site all we were greeted with was a simmple `Hello World` message. As I learned from my experience with `bashed` a good first step is to enumerate. We decided to use the traditional tools; nmap's http-enum script and good old `dirb`.

Attempting to enumerate `10.10.10.68` we discovered that first of all ports 80 and 22 were open. Further scans revealed little else to go off here. Niether `http-eum` nor `dirb` uncovered any new directories. We decided to try our luck with the open `ssh` port. After a few basic guesses at the username/password we decided, considering this was classed as an easy-medium machine that there had to be a more obvious way to gain a user shell.

Navigating back the page presented by the webserver we discovered that the page source contained a hint: `10.10.10.68/nibbleblog/`. Back to enumeration! 

This time `dirb` graced us with a more promising list of pages and directories. Amongst these was `nibbleblog/admin.php`. Bingo!

The admin page presented us with a a login. Our first approach was attempting to use `Burp Suite` to brute-force the login credentials. But as a rule of thumb we decided to try some basic username/password combinations....

```
admin : admin
user : user
test : test
admin : welcome123
nibbles : nibbles
admin : nibbles
```

Jackpot!  `admin : nibbles` gave us access to the administrator web console. 

Seeking any vulnrabilities with `nibbleblog` we uncovered an exploit in the way that it uploads and hosts images. Using this it is possible to upload and execute a php web shell. We decided to use metasploit's pre-built `nibbleblog` exploit. This gave us a shell within seconds, only requiring credentials and target.

Using this shell it was easy to find the user flag in the `/user/nibbles/user.txt` file. The root flag required a little more work.

Using the tricks i picked up from `bashed` I began trying to find a path to `privesc`. FIrst step was to see if there are any intersing things we can use `root` for without a password. 

`sudo -l` revealed that there is a file `monitor.sh` that can be execute as root without requiring a pass. Bingo, this is our guy. Navigating to the directory and having a brief look at the contents of the script, it appeared to run some basic monitoring checks, cpu, memory, pagefile and output the results to a text file. We decided to replace the contents of this `monitor.sh` file with a small script that would simple read and echo the contents of the `/root/root.txt` file.  Running the script easily gave us the `root` flag.