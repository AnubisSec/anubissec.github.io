---
layout: single
title: Using Ansible and Terraform to Build Red Team Infrastructure
date: 2021-09-06
classes: wide
tags:
    - Automation
    - Infrastructure
    - Ansible
    - Terraform
---


It has been a long time for any type of blog or content, but I'm happy to writing again.

This time around, I will show some of the things I've learned when it comes to automating and standing up Red Team Infrastructure. I've always been interested in making infrastructure as least identifiable as possible. There is a lot of debate as to *if* or *why* Red Teams should make their infrastructure obfuscated and put in all this effort of redirectors and domain fronting. I won't be getting into the philosophy of all this, but at the end of the day, I'm being paid to emulate threat actors, and help our defenders detect these actors. More advanced threat actors **will** use a series of complex infrastructure, and I want to help protect our network from those kind of people. 

Enough set up, let's get to building! :)

First Steps
--------------

One of the biggest things Red Teams will have to try and figure out is *what* kind of infrastructure do they want/need. What kind of C2 infrastructure are you using, what kind of operations are you going to be conducting, HTTPS or DNS (or both), and so on. For my first round of testing/learning, I kept it very simple. See the diagram below for the netflow of what this automation will set up:

![](/assets/images/AutoInfra/NetDiagram.png)

I promise I'm not 12 based on that diagram, but it gets the point across! The target will run an implant that will call out to a domain, which the DNS entry points to `hop1`, which just is hosting `nginx` using the Proxy Protocol (which we will dive into soon). The Proxy Protocol just sends our data over to hop2, which will terminate our TLS comms and have a redirect rule to send the traffic to our C2 server.

This is a fairly simple redirector set up, but instead of just having two `nginx` hosts using redirector rulesets, we utilize the Proxy Protocol as well as a few other tricks which we will dive into. Of course with this template, you can add as many `hops` as you'd like, but just realize that's more overhead with very little reward. 

And now we will dive into actually automating setting all of this up and configuring it so that with a couple of clicks, you will have your very own infrastructure to use on operations!

Setting up Infrastructure
--------------

So the first challenge we need to tackle is what service to use to host our infrastructure, and how to stand it up programmatically. For my requirements, I went with [GCP](https://cloud.google.com/compute), but you can use any service that supports Terraform. 

*This walkthrough assumes you already have your C2 server stood up and all the domain information for it and everything established, but that part can certainly be added to the automation process.*

We will go through building our Terraform script step by step:

```terraform
# Networking to get public IP
resource "google_compute_address" "static" {
    count = 2
    name = "redir-address-${count.index}"
    project = "<YOUR_PROJECT_NAME>"
    region = "us-east4"
}
```

This code snippet set up the static IP information, just telling GCP to allocate 2 static public IP addresses, and naming them `redir-address-0` and `redir-address-1`. We set two different names so that we can have finer control of which one goes to what instance and all that if needed. 

```terraform
# Machine startup
resource "google_compute_instance" "redir" {
    count = 2
    name = "ephemeral-redir-${count.index}"
    machine_type = "f1-micro"
    tags = ["redir"]
    project = "<YOUR_PROJECT_NAME>"
    zone = "us-east4-c"

    boot_disk {
        initialize_params {
            image = "ubuntu-os-cloud/ubuntu-2004-lts"

        }
    }
```

This next snippet above stands up 2 server instances, again naming them `ephemeral-redir-0` and `ephemeral-redir-1` so we have a higher sense of control. I also tag them as `redir` and give them an Ubuntu 20.04 image to boot with. 

```terraform
    # Set ephermal IP and put in the redir subnet/access list 
    network_interface {
        network = "redir"
        access_config {
        nat_ip = element(google_compute_address.static.*.address, count.index)
        }
    }
```

This snippet sets up the network interfaces for each instance, taking the index of each one and assigning them available static IP's.

```terraform
# Display the IP's of each hop, mainly to assit with adding IP to google domains
output "gcp_ips" {
    value = [
        for gcp in google_compute_instance.redir:
            "${gcp.name} : ${gcp.network_interface.0.access_config.0.nat_ip}"
    ]
}
```

Finally, we define output for Terraform to display which instance has what static IP address, so that we can define our DNS records accordingly.

All together, our terraform script looks like this:

```terraform
# Networking to get public IP
resource "google_compute_address" "static" {
    count = 2
    name = "redir-address-${count.index}"
    project = "<YOUR_PROJECT_NAME>"
    region = "us-east4"
}

# Machine startup
resource "google_compute_instance" "redir" {
    count = 2
    name = "ephemeral-redir-${count.index}"
    machine_type = "f1-micro"
    tags = ["redir"]
    project = "<YOUR_PROJECT_NAME>"
    zone = "us-east4-c"

    boot_disk {
        initialize_params {
            image = "ubuntu-os-cloud/ubuntu-2004-lts"

        }
    }

    # Set ephermal IP and put in the redir subnet/access list 
    network_interface {
        network = "redir"
        access_config {
        nat_ip = element(google_compute_address.static.*.address, count.index)
        }
    }

}

# Display the IP's of each hop, mainly to assit with adding IP to google domains
output "gcp_ips" {
    value = [
        for gcp in google_compute_instance.redir:
            "${gcp.name} : ${gcp.network_interface.0.access_config.0.nat_ip}"
    ]
}
```

Great, now we have a Terraform script that will stand up two servers for us, and assign them public IP addresses, and print out their info. Now what?

Note: *Once the IP's get printed when running Terraform, you'll want to quickly grab the `hop1` public IP address and edit whatever DNS service you use to point to your domain name of choice*

Next, we will utilize Ansible in order to manage our newly stood up servers and push out configurations to them!

Setting up Configuration Management
--------------

So ideally, everything would be able to be done under one tool (such as Terraform), but sadly I have been learning that Infra/Dev Ops is a balancing act of what different tools to use to do very specific functions. 

As such, we will now bring in another tool, called Ansible. Ansible will, in our case, use SSH to log into our "inventory" (inventory being our GCP instances), and execute a series of commands and upload configuration files. Alongside Ansible, we will utilize a service called Jinja2, which will help dynamically edit our configuration files. 

Let's start breaking down these steps!

Ansible Inventory
--------------

First thing we need to set up is our "inventory" file. This is a file that Ansible uses to know what servers it can try to connect to and do certain tasks. There is a plugin for Ansible to work with GCP a lot more symbiotically called `gcp_compute`. 

Here is the very simple inventory file contents:

```yaml
---
  plugin: gcp_compute
  projects:
    - <YOUR_GCP_PROJECT_NAME_HERE>
  auth_kind: application

  # takes each instance and assigns them a group name so ansible can differentiate them
  groups:
    hop1: "'ephemeral-redir-0' in name"
    hop2: "'ephemeral-redir-1' in name"
```

So we have it authenticate using our shell environment to our GCP project, and then we set up our `groups`.

This took me a while to refine, since it's difficult to fully control how Ansible will execute tasks on certain GCP instances. What this syntax does, is it will assign each GCP instance to it's own `group` and the way it sorts this is based on the instance name we defined further up in Terraform. So if it has `ephemeral-redir-0` name in GCP, it will be assigned to `hop1` group in Ansible. The same applies to `ephemeral-redir-1` with `hop2` as well.

And that's it :), we now have our inventory file set up! Let's move on to the main Ansible yaml file!

Ansible Playbook
--------------

Now that we have our inventory all set up, we can start defining our "plays". Plays are just different actions Ansible will take on hosts. Let's take a look at the first "play" we will define:

```yaml
# This pertains to all the hosts in the inventory list
- hosts: all
  become: yes
  tasks:
  - name: "apt update"
    apt:
        update_cache: yes
        cache_valid_time: 3600

  - name: "Install nginx"
    apt:
        name: ['nginx']
        state: latest
```

In the `hosts` parameter, we define `all` which means these commands will be run on both servers. We update our `apt` packages and install `nginx` on each box. These are basically the only two things that will be the same on each host, so that's all we will define for now.

Now we move onto `hop1`:

```yaml
- hosts: hop1
  become: yes
  tasks:

      - name: "Copy Template nginx conf 1"
      template:
        src: hop1.conf.j2
        dest: /etc/nginx/stream_redirect.conf
        owner: root
        group: root
        mode: '0644'
    
    - name: "Add stream conf to nginx conf"
      ansible.builtin.shell:
        cmd: "echo 'include /etc/nginx/stream_redirect.conf;' | sudo tee -a /etc/nginx/nginx.conf"
      notify: restart_nginx

  # If I don't put ansible.builtin.service, this literally doesn't work
  handlers:
    - name: restart_nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

All this "play" does above is, copy over a local file (which we will analyze after this), append a string to our `nginx.conf` file on `hop1` and then restart nginx.

Let's take a look our `/etc/nginx/stream_redirect.conf` template:

```
stream {
        server {
                listen 443;
                proxy_pass {{ groups['hop2'] | first }}:443;
                proxy_protocol on;
        }
}
```
That's it, ha! All our nginx server will be doing is listening on port 443, and then pass all traffic that goes to port 443 over to our second hop, `hop2`. The syntax seen here, `{{ groups['hop2'] | first }}` is utilizing jinja2 to pull variables dynamically when we run Ansible. the syntax inside the curly brackets just pulls the public IP address of our second hop. 

So we basdically just take that config above, and push it to our first hop server, and restart nginx. That server is now configured and ready to go! Now we move on to `hop2` play.

If you want to learn more about how nginx `Stream Proxy Protocol` works, check out this [blog](https://0xda.de/blog/2020/02/red-team-proxy-protocol-nginx/) by [@0xdade](https://twitter.com/0xdade), which helped me come up with this half of my idea!


So this part is a bit of a hassle right now and will be updated in the future. To start this play, before running anything you need to use `certbot` and get your SSL certficate information ahead of time, and put it in the current working directory of your playbook. This is becasue we will but uploading our `fullchain` and `privkey` keys to our second hop manually, since running certbot through Ansible is verrrrrry broken (at least the way I was doing it), *as well as the fact you can only request SSL certificates for a domain a certain amount of times per week.* Though you shouldn't need to run certbot multiple time to run into this caveat, during my testing and troubleshooting I was, so this is how I have it configured...

Let's take a look:

```yaml
# Before restarting nginx, we need to do certbot stuff
- hosts: hop2
  become: yes
  tasks:

    - name: "Create letsencrypt directory"
      ansible.builtin.file:
        path: /etc/letsencrypt/live/{{ domain }}
        state: directory
        

    - name: "Copy over full chain"
      ansible.builtin.copy:
        src: fullchain.pem
        dest: /etc/letsencrypt/live/{{ domain }}/fullchain.pem
    
    - name: "Copy over privkey"
      ansible.builtin.copy:
        src: privkey.pem
        dest: /etc/letsencrypt/live/{{ domain }}/privkey.pem

    - name: "Copy Template nginx conf 2"  
      template:
        src: hop2.conf.j2
        dest: /etc/nginx/sites-enabled/default
        owner: root
        group: root
        mode: '0644'
      notify: restart_nginx

  handlers:
    - name: restart_nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

So as you can see, it's effectively the same as the first hop, `hop1`, but we just upload the SSL certficiate stuff as well. Also note the `{{ domain }}` variables used within this play. This variable is defined at run time, using the `--extra-vars "domain=<domain_name_here>"` when running ansible CLI.

Let's take a look at the nginx configuration for our second redirector:

```
server {
        listen 443 ssl proxy_protocol;
        listen [::]:443 ssl proxy_protocol;

        ssl_certificate /etc/letsencrypt/live/{{ domain }}/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/{{ domain }}/privkey.pem; # managed by Certbot
        ssl_session_cache shared:le_nginx_SSL:1m; # managed by Certbot
        ssl_session_timeout 1440m; # managed by Certbot

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # managed by Certbot
        ssl_prefer_server_ciphers on; # managed by Certbot

        ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256 ECDHE-ECDSA-AES256-GCM-SHA384 ECDHE-ECDSA-AES128-SHA ECDHE-ECDSA-AES256-SHA ECDHE-ECDSA-AES128-SHA256 ECDHE-ECDSA-AES256-SHA384 ECDHE-RSA-AES128-GCM-SHA256 ECDHE-RSA-AES256-GCM-SHA384 ECDHE-RSA-AES128-SHA ECDHE-RSA-AES128-SHA256 ECDHE-RSA-AES256-SHA384 DHE-RSA-AES128-GCM-SHA256 DHE-RSA-AES256-GCM-SHA384 DHE-RSA-AES128-SHA DHE-RSA-AES256-SHA DHE-RSA-AES128-SHA256 DHE-RSA-AES256-SHA256 EDH-RSA-DES-CBC3-SHA"; # managed by Certbot

        root /var/www/html;

        index index.php index.html index.htm index.nginx-debian.html;

              if ($http_user_agent != "<REDACTED>") {
                return 302 https://<REDACTED>.com;
              }

        server_name {{ domain }};

        location / {
                try_files $uri $uri/ @c2;
       }

              location /<REDACTED> {
                try_files $uri $uri/ @c2;
        }

        location /<REDACTED> {
                try_files $uri $uri/ @c2;
        }

        location @c2 {
                proxy_pass https://<REDACTED>;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
}
```

Obviously I redacted a lot of info in this config as to not give away any IOC's or special sauce that my teams uses...but this is enough to help get others started.

So first the most notable thing to insert is:
```
listen 443 ssl proxy_protocol;
        listen [::]:443 ssl proxy_protocol;
```

Note the `proxy_protocol` at the end of each of these lines. This will allow our second nginx redirector aware of proxy_protocol traffic, which is what will be coming from our first redirector. 

Next we set up all the SSL certficiate information, and a simple rule to filter out traffic based on a particular user agent. 

We then have some pre-defined locations on our second hop that will redirect traffic to our C2 server, which is defined under the `location @c2` line. The `proxy_pass` line that contains our redacted c2 information can be automated using jinja as well, but for my team's case, it was just easier to hardcode it, since this source code lives in a private repository only accessible by our team. 

So obviously filling in the `REDACTED` gaps into this configuration, you should be good to go for this play to succesfully run!

One thing that I am not show casing on here due to sensitive information and brevity, is firewall rules. You will need to automate that part to allow target traffic into `hop1`, and of course for `hop1` to talk to `hop2`. 

Finally, I have created a small bash script to run both terraform and ansible without having to manually insert too many things:

```sh
domain="$1"

terraform apply -auto-approve

echo -e "\n[+] Sleeping 30s to let SSH start up..."
sleep 30

if [ "$1" == "" ]; then
        echo "[!] You didn't supply domain... continuing"
        read -r -p "Are you sure you want to continue without a domain? This will set up infra, but not configure redirs [y/N] " response
        if [[ "$response" =~ ^([yY][eE][sS]|[yY])$ ]]
        then
                ansible-playbook -i inventory/gcp.yaml nginx_hops.yml
        fi
else
        ansible-playbook -i inventory/gcp.yaml nginx_hops.yml --extra-vars "domain=$1"
```

As for your file structure for this project, you will want:

```
main/
├─ inventory/
│  ├─ gcp.yaml
├─ templates/
│  ├─ hop1.conf.j2
│  ├─ hop2.conf.j2
├─ fullchain.pem
├─ privkey.pem
├─ nginx_hops.yaml
├─ gcp.tf
    
```

Final Thoughts
--------------

Of course, this set up isn't perfect, and I am actively trying to make this a lot better. Such as automating the DNS entry issue, as well as the certbot/certificate pipeline as well. Perhaps I will make a follow up blog when I make these optimizations.

Finally, I would like to give a special thanks to these folks:
1. [bluescreenofjeff](https://bluescreenofjeff.com/2018-04-12-https-payload-and-c2-redirectors/) for this awesome post
2. Once again, [0xdade](https://0xda.de/blog/2020/02/red-team-proxy-protocol-nginx/) for this amazing idea
3. All the struggling devops peeps posting on stackoverflow that shared my pain :)



~ Hack the planet! ~
