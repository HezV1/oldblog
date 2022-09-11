---
author: Hez
tags: walkthrough nibbles HTB
---
**Goal:** Give a walkthrough of the HTB Starting Point boxes shooting for a layman-level technical explanation.

HTB has one of the best learning platforms for offensive security, so I'll use it as a guide for brushing up on my pentesting chops. I'll make writeups for each box to document my work, structure my thoughts, and maybe provide some value for a future reader. So, starting with the first box, nibbles.   

# Nibbles
```
Creator - mrb3n
Operating System - Linux
Difficulty - Easy
User Path - Web
Privilege Escalation - World-writable File / Sudoers Misconfiguration
Ippsec Video - https://www.youtube.com/watch?v=s_0GcRGv6Ds
Walkthrough - https://0xdf.gitlab.io/2018/06/30/htb-nibbles.html
```

## Recon
Recommended scan from the HTBA module is `nmap -sV --open -oA nibbles_initial_scan <ip address>` doing a *version scan* `-sV` to have nmap output what version of each service is listening on every `--open` port. Saving the result with `-oA` is also a good habit.

Nmap, by default, only scans the top 1,000 ports. The reading gives us a quick command to see what those ports are if we're curious, `nmap -v -oG -` runs a scan with no scan options, so it just lists the ports.

Very cool, nmap.

Onto the actual scan

```bash
$ nmap -sV --open -oA nibbles_initial_scan 10.129.200.170
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-09 03:32 CDT
Nmap scan report for 10.129.200.170
Host is up (0.11s latency).
Not shown: 924 closed tcp ports (conn-refused), 74 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.76 seconds

```

So we have OpenSSH listening on port 22 and an Apache server listening on port 80. We also have our files from the scan in `.gnmap`, `.nmap`, and `.xml` formats. 

HTB Academy also recommends we do full TCP port scan with `nmap -p- --open -oA nibbles_full_tcp_scan <ip address>`. using `-p-` to specify all ports. While this might seem redundant, I think HTBA is trying to ingrain a habit of running longer scans in the background, which is good. The concept transfers over to other tools as well. If we can configure it, we should use shorter tasks while actively working and later run longer background tasks while doing other active work.

In this case, the full port scan doesn't find anything more.

Next up - NC for some banner grabbing

```shell
$ nc -nv 10.129.200.170 22
10.129.200.170 22 (ssh) open
SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.2
```

Confirms what nmap told us about the SSH server.

```shell
$ nc -nv 10.129.200.170 80
10.129.200.170 80 (http) open
```

Confirms that port 80 is open, but we don't get a banner out of the web server.

Now that we know we're going to be focusing on ports 22 and 80, we can go ahead and run a more detailed nmap scan. `nmap -sC -p 22,80 -oA nibbles_script_scan <ip_address>` will run a nmap script scan with `sC` and just on those 2 ports. This will use a default set of scripts built into nmap to do more detailed service enumeration.

```shell
$  nmap -sC -p 22,80 -oA nibbles_script_scan 10.129.200.170
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-10 03:26 CDT
Nmap scan report for 10.129.200.170
Host is up (0.11s latency).

PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http
|_http-title: Site doesn't have a title (text/html).

Nmap done: 1 IP address (1 host up) scanned in 4.69 seconds
```

From this, we don't get much extra. We get the ssh server's public host key and not much else.

Last step for enumeration, we can use a specific nmap script that isn't included in the default scripts when we use `-sC`. `nmap -sV --script=http-enum -oA nibbles_nmap_http_enum <ip_address>` running a version scan with the `http-enum` script.

```shell
$ nmap -sV --script=http-enum -oA nibbles_nmap_http_enum 10.129.200.170
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-10 03:33 CDT
Nmap scan report for 10.129.200.170
Host is up (0.11s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.70 seconds

```

The script only added the `http-server-header` line, which doesn't tell us anything we didn't already know. 

Regarding our recon phase, we've got a good amount of info about what is on the host. So the next step is more recon. We will dive into the identified services and do more detailed recon, starting with the web server.
