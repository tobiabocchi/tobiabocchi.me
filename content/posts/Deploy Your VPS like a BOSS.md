---
date: "2024-06-05T12:09:44+02:00"
draft: false
title: "Deploy Your VPS Like a BOSS"
description: "Using Ansible to deploy websites and a mail server on your VPS"
summary: "Deploy websites and a mail server on your VPS using Ansible"
cover:
  image: "/img/posts/deploy_like_a_boss/deploy_like_a_boss.jpg"
---

It's been a while since I bought my first vps, I mainly use it to host a few websites
and a mail server and I tought, since I am now looking to start a carreer as DevOps
it would be a good exercise to try and automate the deployment of these services
using Ansible.

## Overview

So far I manage 3 websites:

1. [My personal blog](https://tobiabocchi.me): you're looking at it right now :)
   referred to as `042007.xyz` in this article
2. [A diary for my trips](https://leggendiario.fun): referred to as `0011234.xyz`
   in this article
3. [My parent's agritourism's website](https://lasiorenna.it): referred to as `628496.xyz`
   in this article

I bought the three `.xyz` domains just for testing purposes before I deployed everything
in production. In case you didn't know, 6-10 digits `.xyz` domains are the cheapest
available right now, they cost about a dollar per year

> ⚠️ Nerd Alert:
> In case you were wondering, 6, 28 and 496 are the first three [**perfect numbers**](https://en.wikipedia.org/wiki/Perfect_number)

- My **current setup** serves the static websites using `nginx` and hosts a mail
  server setup using Luke Smith's [emailwiz](https://github.com/LukeSmithxyz/emailwiz)
  for accounts using the blog's domain.
- My **goal** is to automate the deployment of these 3 websites plus a mailserver
  (mail.042007.xyz) which will also handle info@domain for the other two.
- This is the **plan**: [nginx](https://nginx.org) will serve the files for the static
  websites, which will implement `https` using letsencrypt certificates, the mailserver
  will run in a docker container using [DMS](https://github.com/docker-mailserver/docker-mailserver)
  (Docker-Mail-Server)

All of this will be configured through [Ansible](https://ansible.com), in order
to make the whole process as automated and repeatable as possible. So when I'm broke
and need to get a cheaper vps provider for hosting my websites I can get them back
up in no time.

## Ansible

Ansible is an **automation tool**, in this case we will leverage its functionalities
to set up a vanilla Debian 11 vps to serve our websites and act as a mail server.
Before we get started writing our automation tasks we need to set up a few things.

Let's create a folder for holding all our files, we'll call it `ansible_vps`, then
inside this folder we create a python environment, activate it, upgrade pip and
install ansible!

```sh
mkdir ansible_vps
cd ansible_vps
python3 -m venv venv
source venv/bin/activate
pip install -U pip ansible
```

### ssh

If you already have a working ssh key with your vps you can skip this part. Just
make sure you also have `~/.ssh/config` configured correctly.

Since ansible uses `ssh` to connect to and run commands on remote hosts it's best
practice to create a ssh key which we'll use for this purpose:

```sh
ssh-keygen -f ~/.ssh/ansible -P ""
```

In case you want to password protect your key, just omit the `-P ""` and you will
be prompted for a password. Just know that if your key is password protected you
can set up SSH agent to avoid retyping passwords in your current terminal session
doing something like this:

```sh
ssh-agent bash
ssh-add ~/.ssh/ansible
```

Now we can edit `~/.ssh/config` to instruct ssh on which key to use when connecting
to our vps:

```sshconfig
Host 042007.xyz
    User root
    Hostname 123.123.123.123 # vps public ip
    IdentityFile ~/.ssh/ansible
    Port 22
```

Ultimately we need to authorise our key to be used on the vps:

```sh
ssh-copy-id -i ~/.ssh/ansible 042007.xyz
# it will ask the root password and then copy the identity file to the vps
```

> Note that to be able to do this the ssh server on the vps must allow root login
> we will disable it shortly for security reasons. In other words, `/etc/ssh/sshd_config`
> on your vps should contain the line `PermitRootLogin yes`

### `inventory.ini`

The first thing we need is a `inventory.ini` which will specify the targets for
our automation scripts. In here we can also specify custom variables which we'll
use later.

```ini
# inventory.ini
[vps]
042007.xyz hostname=mail.042007.xyz public_ip=123.123.123.123
```

We can verify the file is parsed correctly using:

```sh
ansible-inventory -i inventory.ini --list
# and test our configuration is working by pinging our vps:
ansible vps -m ping -i inventory.ini
```

## Automation

Ok, it's time to get started writing our automation scripts! The main file from
which we'll reference them is `playbook.yml`, here is its content:

```yml
# playbook.yml
- name: Setup vps
  hosts: vps # defined in inventory.ini
  roles:
    - essentials
    - security
    - nginx
    - mail
```

As we can see our playbook will run a few **roles**:

- **essentials**: initial configuration steps, setting hostname and installing packages
- **security**: security related configuration, mainly firewall
- **nginx**: everything related to our static websites
- **mail**: the mail server configured using Docker-Mail-Server

### Essentials

Each role will reside in a directory, where we'll specify what that role does.
We can start by creating the directory for the **essentials** role with a **tasks**
subdirectory where we'll specify the tasks for this role.

Within `tasks` we create `main.yml` which will set the hostname on the vps, create
a directory for our scripts and reference `packages.yml` in the same directory
as `main.yml`

```sh
mkdir -p essentials/tasks
touch essentials/tasks/{main.yml,packages.yml}
tree essentials
# essentials
# └── tasks
#     ├── main.yml
#     └── packages.yml
```

Here is the content of our newly created tasks:

```yml
# FILE: main.yml
- name: set hostname
  ansible.builtin.hostname:
    name: "{{ hostname }}" # defined in inventory.ini
- include_tasks: packages.yml
- name: create script directory
  ansible.builtin.file:
    path: /root/scripts
    state: directory
# EOF
# FILE: packages.yml
- name: install packages
  ansible.builtin.apt:
    update_cache: yes # apt update before installing
    name:
      - bat
      - build-essential
      - fd-find
      - fzf
      - git
      - htop
      - magic-wormhole
      - man-db
      - manpages
      - net-tools
      - ripgrep
      - rsync
      - speedtest-cli
      - sudo
# EOF
```

All of the above should be pretty self explanatory, note thate when using `{{ }}`
within the `yml` files we can reference the variables we specified in `inventory.ini`

In other words, the essentials role, sets the `hostname` for our server, installs
a few packages and create a `/root/scripts` directory which we'll use to store scripts
later on.

At this point we can comment out the roles we have not setup yet in `playbook.yml`
by just putting a `#` in front of them (all but essentials) and try out our playbook
like so:

```sh
# NOTE: activate the python virtual environment for using ansible-playbook
ansible-playbook playbook.yml
```

### Security

Now that we have a working ansible setup we can start working on our playbook to
implement different features, first of all let's start with **security**.

We are going to secure network connections to the vps via a packet filtering firewall:
`iptables`. The file structure for this role will look like this:

```text
security              # parent folder for the role
├── files
│   └── rules.v4      # firewall rules
│   └── sshd_config
└── tasks
    ├── iptables.yml
    └── main.yml
```

Here is the content of the `yml` files:

```yml
# FILE: tasks/main.yml
- include_tasks: iptables.yml
- name: load sshd_config
  ansible.builtin.copy:
    src: files/sshd_config
    dest: /etc/ssh/sshd_config
    backup: yes
- name: Restart sshd
  ansible.builtin.service:
    name: sshd
    state: restarted
    enabled: yes
# EOF
# FILE: iptables.yml
- name: ensure iptables is installed
  ansible.builtin.apt:
    update_cache: yes
    name:
      - iptables
      - iptables-persistent
- name: copy firewall rules
  ansible.builtin.copy:
    src: files/rules.v4
    dest: /etc/iptables/rules.v4
    owner: root
    group: root
    mode: "644"
    backup: yes
- name: ensure iptables is restarted and enabled at boot
  ansible.builtin.service:
    name: iptables
    state: restarted
    enabled: yes
# EOF
```

Basically this role after making sure `iptables` is installed, copies a file to
the vps, `rules.v4` which contains the rules for our firewall:

```sh
# rules.v4
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
# interface related rules
-A INPUT -i lo -j ACCEPT -m comment --comment "Allow incoming loopback traffic"
# connstate related rules
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT -m comment --comment "Allow related and established traffic"
# protocol related rules
-A INPUT -p icmp -j ACCEPT -m comment --comment "Allow icmp traffic"
# port related rules
-A INPUT -i eth0 -p tcp --dport 22  -j ACCEPT -m comment --comment "ssh"
-A INPUT -i eth0 -p tcp --dport 25  -j ACCEPT -m comment --comment "SMTP"
-A INPUT -i eth0 -p tcp --dport 80  -j ACCEPT -m comment --comment "http"
-A INPUT -i eth0 -p tcp --dport 143 -j ACCEPT -m comment --comment "IMAP"
-A INPUT -i eth0 -p tcp --dport 443 -j ACCEPT -m comment --comment "https"
-A INPUT -i eth0 -p tcp --dport 465 -j ACCEPT -m comment --comment "SMTPS"
-A INPUT -i eth0 -p tcp --dport 587 -j ACCEPT -m comment --comment "SMTP Submission"
-A INPUT -i eth0 -p tcp --dport 993 -j ACCEPT -m comment --comment "IMAPS"
# reject the rest
-A INPUT -p tcp -j REJECT --reject-with tcp-reset
-A INPUT -p udp -j REJECT --reject-with icmp-port-unreachable
-A INPUT -j REJECT --reject-with icmp-proto-unreachable
COMMIT
```

The rules are grouped by what they filter for: interface, connection state, protocol,
port. Using a policy of **ACCEPT** for `OUTPUT` and `FORWARD` chains allows all
the traffic "exiting" or "transiting" the vps, for the `INPUT` chain we set a policy
of **DROP** and only allow a few things:

1. Loopback traffic, when the vps tries to reach itself through the loopback interface
2. Packets with state of `RELATED` or `ESTABLISHED`
3. `icmp` protocol packets, for example to `ping` the vps
4. Traffic to ports 22 for `ssh`, 80 for `http` and 443 for `https`
5. Mail server related traffic on ports 25, 143, 465, 587 and 993

After dealing with the firewall, this role also copies to the vps a file named
`sshd_config` which contains the settings for its ssh server:

```sshdconfig
PermitRootLogin prohibit-password # Only allow key auth for root login
ChallengeResponseAuthentication no
PasswordAuthentication no # Disable password authentication entirely
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

This is it for the security role.

### Websites

Now let's focus on the websites that will be hosted on the vps. The web server we'll
use is `nginx`, once again this is the file structure for this role:

```text
nginx
├── files
│   ├── sites-available
│   │   ├── 0011234.conf
│   │   ├── 042007.conf
│   │   ├── 628496.conf
│   │   └── mail.conf
│   └── www
│       └── websites.tar.gz
└── tasks
    └── main.yml
```

We have 4 `.conf` entries, the three domains and mail, the latter is needed just
to fetch openssl certificates for the mailserver, it won't actually serve any website.
Then, inside the folder `www` we find a tar compressed archive containing three
folders (one per website) containing the static website's files.

```yml
# main.yml
- name: install nginx and certbot
  ansible.builtin.apt:
    update_cache: yes # apt update before installing
    name:
      - nginx
      - python3-certbot-nginx
- name: copy configuration files
  ansible.builtin.copy:
    src: files/sites-available/
    dest: /etc/nginx/sites-available
- name: copy websites
  ansible.builtin.unarchive:
    src: files/www/websites.tar.gz
    dest: /var/www/
- name: enable websites
  ansible.builtin.shell: ln -fs /etc/nginx/sites-available/* /etc/nginx/sites-enabled/
- name: disable default website
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
- name: retrieve letsencrypt certificates
  ansible.builtin.command: certbot -m "mail@example.com" --agree-tos --no-eff-email -d "042007.xyz,mail.042007.xyz,628496.xyz,0011234.xyz"
- name: ensure nginx is restarted and enabled
  ansible.builtin.service:
    name: nginx.service
    state: restarted
    enabled: yes
```

> Note: for copying the websites' files we use **unarchive**. This is because
> the copy module is very slow when handling lots of files, so we make a single
> compressed tar archive and move that instead.

The `.conf` files for the websites are very simple:

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name 042007.xyz;
        root /var/www/042007;
        index index.html;
        error_page 404 /404.html;
        location / {
                try_files $uri $uri/ =404;
        }
}
```

Certbot will conveniently take care of setting up `https` and redirect traffic
from `http`.

And just like that we have deployed the websites and set up letsencrypt certificates
for them.

One last thing we need to do for making the websites reachable from the outside
is setting up DNS records, this is my current setup:

- One single `A` record registered for 042007.xyz which points to the public ip
  of the vps
- Two `CNAME` records, one per additional domain (0011234.xyz and 628496.xyz)
  pointing to 042007.xyz, this way, if the public IP of my vps changes (e.g. I
  migrate to a new service provider) I will only need to change the single `A`
  record.

### Mail

Now the last part: mail

As previously said we'll use Docker-Mail-Server to provide mail for all the three
domains.

To satisfy the requirements to be able to send mail that gets accepted by other
providers we need to set a reverse DNS record (`PTR`) pointing to mail.042007.xyz,
this will do the opposite of an `A` record: it will link the public address of our
vps to our mail server domain name. Furthermore an `MX` record is necessary for
email to work properly, we set ours to mail.042007.xyz.

The following is the file structure for the `mail` role in the playbook:

```text
mail
├── files
│   ├── dms
│   │   ├── compose.yaml
│   │   └── mailserver.env
│   ├── dms.sh
│   └── docker.sh
└── tasks
    └── main.yml
```

The two files `compose.yaml` and `mailserver.env` can be retrieved from DMS's repo
on github, then you should customize them to your needs following the guides on
their documentation.

And here is `main.yml`

```yml
- name: install apparmor
  ansible.builtin.apt:
    update_cache: yes
    name:
      - apparmor
- name: copy docker install script
  ansible.builtin.copy:
    src: files/docker.sh
    dest: /root/scripts/
    mode: "744"
- name: Run docker install script
  ansible.builtin.command: /root/scripts/docker.sh
- name: copy dms folder
  ansible.builtin.copy:
    src: files/dms
    dest: /root/
- name: copy dms setup script
  ansible.builtin.copy:
    src: files/dms.sh
    dest: /root/scripts/
    mode: "744"
- name: Run setup script
  ansible.builtin.command: /root/scripts/dms.sh
- name: ensure Docker is started and enabled at boot
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes
```

This task is quite simple, it copies and run two scripts on the vps, one to install
docker, and the second one to deploy a DMS container:

```sh
#!/bin/sh
# FILE: docker.sh
# Add Docker's official GPG key:
apt update
apt install -y ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |
  tee /etc/apt/sources.list.d/docker.list >/dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Now that docker is installed on our vps, this is the second script that deploys
the dms container, creates a few users and sets up dkim

```sh
#!/bin/sh
# FILE: dms.sh
cd /root/dms || exit
docker compose up -d
sleep 5
docker exec -it mailserver setup email add info@042007.xyz  supersecretpassword1
docker exec -it mailserver setup email add info@0628496.xyz supersecretpassword2
docker exec -it mailserver setup email add info@0011234.xyz supersecretpassword3
docker exec -it mailserver setup config dkim
docker compose down && docker compose up -d
```

Now all that's left to do is to set up a few DNS records for mail:

1. `DKIM` basically authenticates the sender using a public/private keypair, the
   private key is stored on our vps and was created by the script we just run, the
   public key will be stored in a `TXT` DNS record, its content can be retrieved
   in `/root/dms/docker-data/dms/config/opendkim/keys/042007.xyz/mail.txt`, and
   similarly for other domains as well.
2. `SPF`: Sender Policy Framework is a way for a domain to list the servers they
   send emails from. In this case we set a `TXT` record that allows us to send mail
   only from the server indicated by our mx record: `v=spf1 mx -all`
3. `DMARC`: Domain-based Message Authentication Reporting and Conformance tells
   a receiving email server what to do after checking SPF and DKIM. Following DMS
   instructions we set it to:

   ```text
   v=DMARC1; p=reject; sp=reject; fo=0; adkim=r; aspf=r; pct=100; rf=afrf; ri=86400; rua=mailto:dmarc.report@tobiabocchi.me; ruf=mailto:dmarc.report@tobiabocchi.me
   ```

All these records can be checked using `dig`:

```sh
dig +short TXT 042007.xyz
dig +short TXT _DMARC.042007.xyz
dig +short TXT mail._domainkey.042007.xyz
```

## Conclusion

That's it! Now we should be able to tear down and bring back up our vps in no time!

Congratulations on making it through the maze of setting up your VPS for web hosting
and email! If you've got any lingering questions, feel free to drop me a line at
tobia@tobiabocchi.me

Fingers crossed my email server is as reliable as it's supposed to be! 😅
