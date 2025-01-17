---
title: "Inclusiveness Writeup"
date: 2025-01-16 12:00:00 -500
categories: [Writeups]
tags: [Proving Grounds]
image: "/media/Inclusiveness/offsec.jpg"
---

# **Inclusiveness Writeup**

> ✋🏻 Hi, In this write-up we are going to pwn one of prouving grounds play mahines on Offensive Security.

## Enumeration and Initial Access

The first step is to scan the machine with nmap

<span style="color:green">$ nmap -sV -sC -p- -A -T4 192.168.111.14</span>

![status](/media/Inclusiveness/nmap.png)

This machine has:

- A web interface running on port 80
- An FTP server on port 21 with anonymous login enabled

After checking the web interface, there isn’t anything juicy there, so let’s move on to fuzzing the directories

![status](/media/Inclusiveness/directs.png)


When looking for the `robots.txt` file on the website, it gave this:  

![status](/media/Inclusiveness/robots.png)

So, to get an answer, we need to change the user agent of the request to that of a search engine.  
Let’s intercept the request in Burp Suite  

![status](/media/Inclusiveness/burp1.png)

Let’s send it to the Repeater

![status](/media/Inclusiveness/burp2.png)

We'll replace it with a user agent, I’m choosing the Googlebot user agent:  

<span style="color:red">Googlebot/2.1 (+http://www.google.com/bot.html)</span>


![status](/media/Inclusiveness/burp3.png)


![fuzz](/media/Inclusiveness/burp4.png)

Awesome! It gave us a path, let’s check it out  

![license](/media/Inclusiveness/secret.png)

Now, as we can see from the link, we can run PHP files. But first, let’s see if we can move around in the file system to access other files  

Let’s check for `/etc/passwd` 

![base64](/media/Inclusiveness/secret_path.png)

And bam, it’s workiiing  

Now we have an FTP service on port 21 with anonymous login allowed, it contains nothing except a `/pub` folder and it’s empty  

After a bit of Googling, we find that the FTP configuration file is located at `/etc/vsftpd.conf`. Let’s see if we can check it  

![base64](/media/Inclusiveness/ftpd.png)

The most important part is this:  

![base64](/media/Inclusiveness/ftp.png)


---

## Lateral mouvement

So, the FTP server is accessible and the path is `/var/ftp/`. Let’s test it out  

I’ve created a `test.php` file, it is a simple page with PHP just to see if it can execute PHP from the FTP server 

```php
<?php
// Display basic information
echo "PHP is working!<br>";

// Display server information
echo "Server IP: " . $_SERVER['SERVER_ADDR'] . "<br>";
echo "Your IP: " . $_SERVER['REMOTE_ADDR'] . "<br>";

// Display current directory
echo "Current Directory: " . getcwd() . "<br>";

// List files in the current directory
echo "<br>Files in this directory:<br>";
$files = scandir('.');
foreach ($files as $file) {
    echo $file . "<br>";
}
?>
```

![404](/media/Inclusiveness/ftp1.png)

Let’s try to access it

![theme edited](/media/Inclusiveness/ftp2.png)

Awesome

Now it’s clear how we can get a reverse shell into the system, just upload the code and access it through the web page 

And here’s the result! 

![revshell](/media/Inclusiveness/revshell.png)

Our reverse shell is working, Awesome

---

## Privilege Escalation

Now it’s time for privilege escalation  

Using LinPEAS I got suggestions for some exploits : 

![linpeas](/media/Inclusiveness/linpeas.png)

I’ve tried them all but none of them worked 

After searching a bit more in the system I found a script in `/home/tom`

![gtfobins](/media/Inclusiveness/script.png)

It checks if you’re the user tom and then it gives root permissions

To bypass it, I need to trick the system into giving the output as tom when executing whoami

<span style="color:green">echo "echo tom" > whoami <br> chmod +x whoami <br> </span>

Now, to change the target system path

<span style="color:green">export PATH=/tmp:$PATH <br> echo $PATH </span>

Let’s test it out

![root](/media/Inclusiveness/test1.png)

![key3](/media/Inclusiveness/root.png)


And we're done, we’ve successfully compromised the machine

See you in another writeup. ✌️
