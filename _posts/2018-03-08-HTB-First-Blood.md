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

 
###User
It took me far too long to get the user flag. My first step was to use `nmap` to scan the IP of the target machine. This gave me a nice output imforming me that port 80 and was open and showed the supported HTTP methods.


![Nmap Scan](/home/sypher/Repos/v0lch0k.github.io/images/bashed-nmap-1.png  "Nmap Scan")

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

![PHPBashed](/home/sypher/Repos/v0lch0k.github.io/images/bashed-2.png  "PHPBashed")
 
 Inside this directory `phpbash.php` was already pre-loaded. Executing it game me user (`www-data-`) access to the machine. From there it was easy to navigate to the `/www-data@bashed:/home/arrexel/user.txt` file and obtain the `user` flag.