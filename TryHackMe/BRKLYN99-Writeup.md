# [THM-BrooklynNineNine](https://tryhackme.com/room/brooklynninenine) - Maxwell Caron
This room is aimed for beginner level hackers but anyone can try to hack this box. There are two main intended ways to root the box.

Inital `nmap` scan:
```
$ nmap -T4 -A -p- -oN allports 10.10.241.180

# Nmap 7.91 scan initiated Mon May 24 20:38:51 2021 as: nmap -T4 -A -p- -oN allports 10.10.241.180
Warning: 10.10.241.180 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.10.241.180
Host is up (0.16s latency).
Not shown: 65450 closed ports, 82 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.13.5.204
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 16:7f:2f:fe:0f:ba:98:77:7d:6d:3e:b6:25:72:c6:a3 (RSA)
|   256 2e:3b:61:59:4b:c4:29:b5:e8:58:39:6f:6f:e9:9b:ee (ECDSA)
|_  256 ab:16:2e:79:20:3c:9b:0a:01:9c:8c:44:26:01:58:04 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Now that we have our `nmap` scan, there are two ways to root this box, I will be showing both. 

## Path 1: Steganography

First we will look at the web server running on port 80.

### Web App

![web page](https://i.imgur.com/EOcEGSW.jpg)

We see a simple page with nothing but an image, after running a gobuster and nikto, we get nothing,so lets dig a little deeper.

![source code](https://i.imgur.com/Z5FfceX.png)

Looking through the source code of the web page we see a comment that referres to `steganography` as well as showing us the name of the background image so we can easily `wget` it onto our box.

### External

![wget](https://i.imgur.com/ZEfXwub.png)

Now that we have the background image on our box we can start to preform some simple steganography using, first I used `steghide` with an empty password.

![steghide](https://i.imgur.com/7emlCLu.png)

Hmm... it seems to be giving us an error let's try `stegcracker`

![stegcracker](https://i.imgur.com/aX8Dg8A.png)

Looks like `stegcracker` worked perfectly and inside of the hidden file we get `holt`'s password, knowing that `ssh` is open from our `nmap` scan we can attempt to login as `holt`.

### Internal

![ssh](https://i.imgur.com/QjD9oFV.png)

We're in! First we can get our `user.txt`. Now it is time to privesc.

![sudo](https://i.imgur.com/iRUyJvm.png)

First and foremost I always run `sudo -l`, and this time it paid off, we see that we can run `/bin/nano` (the text editor) as any user without needing to supply a password.

![gtfo](https://i.imgur.com/HhpBsJN.png)

Heading over to [GTFOBins](https://gtfobins.github.io/) we can search `nano` and click on the `sudo` tag because we have `sudo` privilages to that command.

![gtfobins2](https://i.imgur.com/f5FeugI.png)

After reading the quick description, we can use these chain of commands to "escalate or maintain privileged access."

```
holt@brookly_nine_nine:~$ sudo /bin/nano
```

Executing this command will bring us into this screen:

![nano](https://i.imgur.com/KwoV6pj.png)

This is just a blank text file, the next two steps are where the exploit begins.

First, pressing `CTRL + r` will give us a `File to insert [from ./]:` field but, looking down underneath the input field, we see that `CTRL + x` is `Execute Command` this sounds *good*, this is also the next step in the GTFOBins post. 

![exploit](https://i.imgur.com/Gux1R7Q.png)

Finally, inputting `reset; sh 1>&0 2>&0` into the `Command to execute` field we will be root!

![root](https://i.imgur.com/3B2q6dP.png)

## Path 2: FTP

### External

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.13.5.204
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
```

From our nmap scan we see that `FTP` is running on port 21 **AND** anonymous login is enabled.

![ftp](https://i.imgur.com/Ofb60XQ.png)

We see one file with the name `note_to_jake.txt` we can use the `get` command to download it onto our box. 

![note](https://i.imgur.com/xJF1wnN.png)

The note is from `amy` telling `jake` that his password is *too weak* the first time reading this I was a bit confused but than I realized that they we're hinting at us to use `rockyou.txt` a wordlist of weak & common passwords. Since there are no login pages on port `80` I assumed that we will be brute-forcing `jake`'s `ssh` login.

![hydra](https://i.imgur.com/JeQpPQj.png)

### Internal

Now that we are on the box we can do some manual enumeration ,once again I start off with `sudo -l`

![sudol](https://i.imgur.com/jcqawRf.png)

We see that we can run the `less` command as any user without supplying a password, lets go to [GTFOBins](https://gtfobins.github.io/) and search for `less`

![lessgtfo](https://i.imgur.com/ef0pMMd.png)

Scrolling down to the `sudo` tab and reading the quick description we see that we can "escalate or maintain privileged access" with the following commands.

```
jake@brookly_nine_nine:~$ sudo /usr/bin/less /etc/profile
```

Running this will use the `less` command as root to edit the `/etc/profile` 

![profile](https://i.imgur.com/SEX7qd4.png)

At the bottom, we see that we have reached the end of `/etc/profile` but we want to replace that line with `!/bin/sh` by simply typing it out.

![binsh](https://i.imgur.com/yw3jy7y.png)

![root](https://i.imgur.com/NZKVBwz.png)

We are now root! You could of also used the sudo privileges of `less` to read the contents of `/root/root.txt` but that's no fun :) 
