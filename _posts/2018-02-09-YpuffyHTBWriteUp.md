Hey there! I've just switched over my old blog website to this new one, so I hope that this will be a better setup and that I will update this one more often!

This first post is going to a write up on the newly retired machine Ypuffy from hackthebox.eu. This was a really interesting (albeit at times, frutrating) box with some unique vectors. From OpenLDAP to fumbling around with BSD. Let's jump into the writeup.



Nmap Output
-----------

```shell_session
root@kali:~/Documents/htb/boxes/Ypuffy# nmap -sV -sC -oA nmap/initial 10.10.10.107
Starting Nmap 7.70 ( https://nmap.org ) at 2019-02-09 20:05 EST
Nmap scan report for 10.10.10.107
Host is up (0.043s latency).
Not shown: 995 closed ports
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.7 (protocol 2.0)
| ssh-hostkey:
|   2048 2e:19:e6:af:1b:a7:b0:e8:07:2a:2b:11:5d:7b:c6:04 (RSA)
|   256 dd:0f:6a:2a:53:ee:19:50:d9:e5:e7:81:04:8d:91:b6 (ECDSA)
|_  256 21:9e:db:bd:e1:78:4d:72:b0:ea:b4:97:fb:7f:af:91 (ED25519)
80/tcp  open  http        OpenBSD httpd
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: YPUFFY)
389/tcp open  ldap        (Anonymous bind OK)
445/tcp open  netbios-ssn Samba smbd 4.7.6 (workgroup: YPUFFY)
Service Info: Host: YPUFFY

Host script results:
|_clock-skew: mean: 1h44m52s, deviation: 2h53m12s, median: 4m51s
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.7.6)
|   Computer name: ypuffy
|   NetBIOS computer name: YPUFFY\x00
|   Domain name: hackthebox.htb
|   FQDN: ypuffy.hackthebox.htb
|_  System time: 2019-02-09T20:10:50-05:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2019-02-09 20:10:49
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .                                                                              
Nmap done: 1 IP address (1 host up) scanned in 23.83 seconds
```

