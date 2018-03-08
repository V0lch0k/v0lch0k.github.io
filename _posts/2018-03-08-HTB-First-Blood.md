---
layout: post
title: Bashed - First-Blood
categories:
- Hack The Box
---

>“As knowing where you're going is preferable to being lost, ask. Rabbit knows a thing or two, and I, myself, don't need a weathervane to tell which way the wind blows. Let your need guide your behaviour; suppress your instinct to lead; pursue Rabbit.”
> 
>―-Cheshire Cat, American McGee's Alice 

This was the first machine I've attempted to pop, the first real exercise in this field. Here I will describe the step I took to get both the `user` and `root` flags for this machine. Following my fumblings, I will detail how I should have proceeded to gain user and root access.

 
##User
It took me far too long to get the user flag. My first step was to use `nmap` to scan the IP of the target machine. This gave me a nice output imforming me that port 80 and was open and showed the supported HTTP methods.


![Nmap Scan]({{"/images/bashed-nmap-1.png" | https://v0lch0k.github.io/images/bashed-nmap-1.png }})

My next step was to browse to the site and see if I can find anything of interest.
Just by browsing the network traffic in the dev console in Chrome, I discovered several directories

- /css
- /php
- /js
- /demo-images

I also followed the links on the site to discover the github page for the `phpbash.php` script for a semi-interactive shell.

My first reaction was that I need to follow the instructions of the `phpbash`. I need to upload the script to the server and execute to access this remote shell.

I spent a long time following dead ends and attempting to understand the JS functions on the page trying to find a way to upload this file to the server. Looking at the screenshots of `phpbash` in action I noticed an `uploads` folder. I then spend a little while attempting to upload the `phpbash.php` file to the uploads folder. Everything from post requests to python scripts. Nothing would work. 

A friend of mine who was also working on the machine at the time gave me a hint of "The owner of the machine does dev work on it". Even after this hint I was still lost. It finally clicked for me when I looked over the screenshots of `phpbash` in action more closely and realized that there must exist a `/dev` directory on the site.

![PHPBASHED]({{"/images/bashed-2.png" | https://v0lch0k.github.io/images/bashed-2.png }})
 
 Inside this directory `phpbash.php` was already pre-loaded. Executing it gave me user (`www-data-`) access to the machine. From there it was easy to navigate to the `www-data@bashed:/home/arrexel/user.txt` file and obtain the `user` flag.
 
 ##Root
 It took me about a week of on and off poking around to get the root flag, and even when I did I went in a round about way to finally get it. At first I wasn't sure how to proceed, Exploring the machine via the web client noticed that there is a `root` folder that I did not have access to. My first assumption was that just like the `user` flag was in `user.txt` the `root` flag must be in a `root.txt` file. 
 
 I attempted to run some simple `find . | grep root.txt` commands which did not yield much information. I decided to try my luck with a reverse shell. After an hours of bumbling around I managed to put together a python script to grant me a reverse shell.
 
 To get the script on the sever I used python's `SimpleHTTPServer` module to host the script on my machine and then using `wget` to download it to the machine through the web client. It wasn't long before I had a reverse shell. However, the reverse shell was still under the `www-data` user, meaning I was not able to do much with it, let alone browse the `root` directory, in which I was certain the flag resided.
 
 Doing some research on `privesc` I encountered a Linux enumeration script; `LinEnum.sh`. The proved to be a powerful tool. I quickly uploaded it to the `tmp` folder on the machine and ran the script. The output was a little more than I expected. Examing the several pages of output I was not completely sure what I was looking for. From my research I understood I need to look for something that was unsual or out of the ordinary, but for someone running this for the first time what was ordinary and what wasn't?
 
 I had also stumbled across a `privesc` script which I quickly ran on the target machine. Once again I recieved several pages worth of output I wasn't quite sure what to do with. I starred at bot the `privesc` and `LinEnum` outputs for several hours trying to find something that could possibly be useful.
 
 I decided to then do some more exploration of the target machine. Exploring the base directory I noticed a `scripts` folder that I also did not have access to as the `www-data` user.  Going back to the `LinEnum` output I had noticed that there was a `scriptmanager` user that did not require a password. 
 
 I attempted to swap users to `scriptmanager` for a good hour. Later to only realise that I could use the `sudo -u scriptmanager <command>` command to execute commands as the `scriptmanager` user. First attempting to gain access to the `root` folder or read the `/root/root.txt` file, then understnading that I now had access to the `scripts` folder via `sudo`. Unfortunately I wasn't sure what to do with this information.
 
 Not knowing the complete ins and outs of how Linux privelages work I decided to write a python script that would look at the `/root/root.txt` and copy the text to a new file in the directory the script was run from. Having brand new access to the the `scripts` folder I thought I would utilize it to house my script. Using `sudo -u scriptmanager` I donloaded the python script to the `scripts` folder. I then attempted to execute the script from that folder only to be greeted with a `permission denied` warning. Slightly disappointed I looked in the `scripts` folder to now be greeted with a file that contained the `root` flag. Not completely sure how I accomplished this I decided to do some research on why this method worked, why even though i was greeted with a `permission denied` output the script was still able to successfully grant me the root flag.
  