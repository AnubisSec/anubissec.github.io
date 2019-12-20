---
layout: single
title: How to Pivot Into Target Network with SSH
date: 2019-12-19
classes: wide
tags:
    - Pentest
    - SSH
    - Tunnel
---


It's been a hot minute, but I thought I would start documenting little things I learn while going through the Offshore labs via [HackTheBox](https://www.hackthebox.eu/). This is a simulated Active Directory forest with simulated users and real life scenarios. Your point is to hack your way though by any means, and get all the flags! It's an added cost to the otherwise free lab set up, but definitely worth the price. 

Without much more preamble, let's get into today's topic: SSH tunnels using sshuttle.


SSHuttle
---------

During an engagement, if you find yourself lucky enough to compromise a host with SSH running that NAT's itself to the internal target network, you have a beautiful pivot point into your target network. Normally the way this would be accomplished would be to set up dynamic tunnels using SSH such as: 
```sh
ssh -D localPort user@remoteHost
```

the `-D` option creates a dynamic port forward when initiating the SSH connection to the remote host. This SSH option allows a user to let the SSH pivot host decide where the packet gets sent to based on the destination. So instead of setting up specific local forwarding rules on the SSH pivot host to route all RDP traffic to a particular host, when you set a dynamic port forward, the SSH pivot host reads the packet sent to it on the dynamic port and decides where to send the traffic as long as there is a route to it. 

While this option is really amazing and useful, it tends to mess up a lot different tools such as `nmap` and especially screws with throwing exploits. A tool that helps get around these hinderences is [sshuttle](https://github.com/sshuttle/sshuttle). This tool will create a seperate tunnel interface on the attacker's machine, similar to a VPN, and then creates a series of `iptable` rules to foward traffic that is intended for the target subnet the attacker can't normally reach. The usage is very simple:

```sh
sshuttle -vr user@webserver targetSubnet/24
```

This will use the SSH host (the webserver above) and create routes through the new interface to reach the target subnet. So when you try to reach any host on the target subnet, it will just automagically route the traffic based on the routes within the new interface. Amazing how it works. 
See below a small diagram on how an attacker could use sshuttle to access a network. We will be following this diagram later on in the blog:

![](/assets/images/OffShore/sshuttle_diagram.jpg)


There will be times too that you need to take advantage of other options from traditional SSH while using `sshuttle`. One issue that I ran into while working within the Offshore labs is that you need to use a private key to SSH into the pivot host. Let's take a test drive with this shuttle shall we?


Use case
---------

I obviously won't get into the specifics of the lab, so I don't ruin how it works, but I will be using the scenario as an example here. The situation is: You have access to a web server, in which you can compromise and escalate your privileges. Once you are the root user, you can get the root SSH key to then gain access when you need it. This pivot host has access to the internal target subnet, which isn't accessible from the outside. Let's try to `curl` an internal web server from the outside, without running any tunnels. 


<script id="asciicast-eB705z6hIMjzB13Kh8DzfUpie" src="https://asciinema.org/a/eB705z6hIMjzB13Kh8DzfUpie.js" async></script>

As you can see in the video above, the `curl` command doesn't respond with any traffic. Now let's set up the `sshuttle` to our pivot host who can reach this IP and see if there is any difference. 

<script id="asciicast-bTtNZlJfiTdCY7eZ3wkB6KAT6" src="https://asciinema.org/a/bTtNZlJfiTdCY7eZ3wkB6KAT6.js" async></script>

Now looking at the video above, you see us running the `sshuttle` command, it setting up the `iptable` rules, and then we curl the same web server address and we get some valid data back. After the `curl` is run, take a note of the window running the `sshuttle` command, it documents the route taken with the source and destination. This is really nice when dealing with multiple tunnels and weird DNS issues, you can pinpoint where exactly in the communication something went wrong.

Also make note, that you see us use the `--ssh-cmd` switch with `sshuttle`, which allows us to run additional SSH arguments within `sshuttle`. You can see us calling the RSA key that is needed for authentication. 

Conclusion
-----------

This is just a brief overview on how you can use `sshuttle` in order to create dynamic and easy to use tunnels into a target network, and not have to rely on dynamic SSH ports, or a mess of local and remote forwarding tunnels that you have to set up by hand. `sshuttle` is a robust tool that you will definitely need to learn if you'd like to do Offshore, and great to know during real engagements.
