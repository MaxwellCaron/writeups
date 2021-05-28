# [THM-GottaCatchEmAll](https://tryhackme.com/room/pokemon) - Maxwell Caron 

This room is based on the original Pokemon series. Can you obtain all the Pokemon in this room?


## External

Inital `nmap` scan:
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 58:14:75:69:1e:a9:59:5f:b2:3a:69:1c:6c:78:5c:27 (RSA)
|   256 23:f5:fb:e7:57:c2:a5:3e:c2:26:29:0e:74:db:37:c2 (ECDSA)
|_  256 f1:9b:b5:8a:b9:29:aa:b6:aa:a2:52:4a:6e:65:95:c5 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Can You Find Them All?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Since we don't have any ssh credentials right now, lets run some scans on the web page running on port `80`.

I'll save you some time after running my inital `gobuster` and `nikto` scans, nothing came back, so looks like it's time for some manual enumeration.

## Web App

![webpage](https://i.imgur.com/NlYY0qd.png)

Looking at the web page, we see that it is just a default `Apache2` page, since we got nothing back from our directory scans, let's look at the source by pressing `CTRL + U`

While scrolling through, we need to look for things that someone might be trying to hide, such as content inside of comments or <hidden> tags.

![comment](https://i.imgur.com/3pqJeNB.png)

The comment above looks unfamiliar from any default `apache2` page I have seen so, lets take a look at the console.

![console](https://i.imgur.com/XiDm39t.png)

We see that there is an array of 10 differeny pokemon.

At this point I looked up if I could some how interract with this array in some weird way or inject it, and no, I was a bit lost until I looked back at the page source and...

![creds](https://i.imgur.com/jV0eYFr.png)

Right above the comment that I was so infatuated about, there are two strage html tags with a colon inbetween them, from experience this normally means an `username:password` combination.

![ssh](https://i.imgur.com/Rtgb2jd.png)

Looks like they worked!

## Internal

![home](https://i.imgur.com/pyINXcg.png)

Taking a look around in our home directory, there isn't anything that really stands out, so I checked the `Desktop` out and, looks like there is a zip file that we can take a look at.

![zip](https://i.imgur.com/ZQ7tpjH.png)

Using the `unzip` command we see that a `grass-type.txt` file was uncompressed. Inside of the file is what seems to be hex-encoded.

Using [CyberChef](https://gchq.github.io/CyberChef/#recipe=From_Hex('Auto')) we can decode the contents and get our first "*pokemon*"!

![homedir](https://i.imgur.com/HsLxUXo.png)

Moving back to the `/home` directory we see that the user `ash` has a home directory as well as `root`'s pokemon but we do not have access to either of these, now we need to find the `water` and `fire` type pokemon

I decided to head over to where the web server is running `/var/www/html`.

![water](https://i.imgur.com/0U2PeBF.png)

There it is! Upon catting the flag out, the flag seems a big off, doesnt really look like any hash or anythin juts a little, *scrambled* once again from experience, I know that we can use an online [Caesar Cipher](https://en.wikipedia.org/wiki/Caesar_cipher) decoder to unscrable the pokemon.

![caesar](https://i.imgur.com/TGBmolF.png)

Now we can submit it, now it's time for the `fire` type pokemon

Since the previous name's of the files had the same structure I decided to use a simple `find` command to see if the pattern continued.

```
pokemon@root:/var/www/html$ find / -name "fire-type.txt" 2>>/dev/null

/etc/why_am_i_here?/fire-type.txt

pokemon@root:/var/www/html$ cat /etc/why_am_i_here?/fire-type.txt

<REDACTED>
```

The pattern did in fact continue and reading the contents of the file we see once again another encoded output, this time it is base64.

![base](https://i.imgur.com/cD8lmIT.png)

By echo-ing the encoded text and piping it into the integraded `base64` command in linux we can see the decoded flag and submit it.

Now it those are all of the flags that the user `pokemon` has direct access to so, now it is time to priv esc

By running a recursive directory search on the `/home` directory we can see if there are any interesting files deep into any file trees, and we get something pretty intersting.

```
pokemon@root:~$ ls -laR /home
```

![lar](https://i.imgur.com/KHuyX7a.png)

The very last output of the command caught my eye, although it is a "cplusplus" file we can still read the file.

![cpp](https://i.imgur.com/MgZjPYd.png)

Inside of the file we see credentials to the `ash` user!

Now that we are a new user we will start our manual enumeration, let's see if we can run any commands with sudo with the `sudo -l` command.

![sudol](https://i.imgur.com/WTBhbKg.png)

Running this command we see that we can run any command as any user, inclusing `root`. Knowing the location of `root`'s pokemon, we can cat-it-out using `sudo`. *(or we can just do `sudo /bin/bash -p` and become `root`)*

![rootflag](https://i.imgur.com/l4NOxJt.png)

Just like that, those are all the flags, thank you for joining me on this writeup :) 
