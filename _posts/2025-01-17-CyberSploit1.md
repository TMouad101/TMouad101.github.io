---
title: "CyberSploit1 Writeup"
date: 2025-01-17 10:00:00 -500
categories: [Writeups]
tags: [Proving Grounds]
image: "/media/CyberSploit1/offsec.jpg"
---

# **CyberSploit1 Writeup**

> ✋🏻 Hi, In this write-up we are going to pwn one of prouving grounds play mahines on Offensive Security, which is CyberSploit1.

## Enumeration and Initial Access

The first step is to scan the machine with nmap

<span style="color:green">$ nmap -sV -sC -p- -A -T4 192.168.215.92</span>

![status](/media/CyberSploit1/nmap.png)

This machine has:

- A web interface running on port 80

After checking the web interface, there isn’t anything juicy there, let's take a look at the /robots.txt file `robots.txt`

![status](/media/CyberSploit1/robots.png)

Since it is encrypted with base64, let's decrypt it

![status](/media/CyberSploit1/pass.png)

Maybe it will be usefull later on

So let’s move on to fuzzing the directories

![status](/media/CyberSploit1/directs.png)

Nothing is usefull here

After checking the source code of the first page

![fuzz](/media/CyberSploit1/user.png)

There’s a username here  

Let’s try logging in with this username since SSH is running 

![fuzz](/media/CyberSploit1/login_1.png)

Awesome, there’s a user with this name

For the password let’s try the output from decrypting the robots.txt

![license](/media/CyberSploit1/login2.png)

And bam, it’s workiiing  

![license](/media/CyberSploit1/login1.png)



---

## Privilege Escalation

Now it’s time for privilege escalation  

Using the CVE-2021-4034 :

![key3](/media/CyberSploit1/root.png)


And we're done, we’ve successfully compromised the machine

See you in another writeup. ✌️
