---
title: "Setting up a static blog (Hugo/Nginx/FreeBSD)"
date: 2021-02-09T18:46:13+05:30
---

## Introduction

> Note: Most of this is a rip-off of the excellent documentation at DigitalOcean. Its compiled here for convenience.

Before starting setup SSH.

## First things

Update the system:

```
$ sudo freebsd-update fetch install
```

Install the following:
* nvim
* git
* bash
* hugo
* nginx

Change the default shell to bash:

```
$ sudo chsh -s bash freebsd
```

### Setting up firewall

Add the following to /etc/rc.conf

```
. . .
firewall_enable="YES"
firewall_quiet="YES"
firewall_type="workstation"
firewall_myservices="22 80 443"
firewall_allowservices="any"
firewall_logdeny="YES"
. . .
```

The third line opens the ports 22, 80 and 443 for SSH, HTTP and HTTPS respectively.

FreeBSD comes with two firewalls, ipfw and pf. To start the firewall:

```
$ sudo service ipfw start
```

To configure how many denials per IP to log, append the following to /etc/sysctl.conf:

`net.inet.ip.fw.verbose_limit=5`


### Setting up the timezone

FreeBSD provides a TUI for this purpose, which is quite nice. To access it run:

```
$ sudo tzsetup
```

We can now enable NTP. To prevent NTP from failing during boot, its recommended to sync it on start. For this run the following:

```
$ sudo sysrc ntpd_enable="YES"
$ sudo sysrc ntpd_enable="YES"
```

Also don't forget to start the service:

```
$ sudo service ntpd start
```

### Setting up Nginx 

Now we need to find the nginx config file. While running the service I figured out the config file was on `/usr/local/etc/nginx/nginx.conf`. But this could be somewhere else, like `/etc/nginx/nginx.conf`, depending on how it was installed.
Once that's figured out, change it to the following: 
```
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        root /home/username/public_html;
        index index.html index.htm;

        # Make site accessible from http://localhost/
        server_name server_domain_or_IP;

. . .
```
Here, it should be noted that the Nginx is configured to serve html from ~/public_html. This is purely for convenience.

> Note: change the `server_domain_or_IP` to the public IP of the website.

### Setting up git hooks

Initially, I had the idea of setting up a cron task to pull from Github and update public_html. A quick google yielded me to git hooks, which was a better way. A little more searching and I found out DigitalOcean has a page describing exactly this! Here's how to set it up: we need to create a bare repo to pull and configure it so that every time we push to the server it runs a script. This can be achieved with post-receive hook.

Clone the repo to the server and then create a bare repo from it:

```
$ git clone --bare ~/blog_repo ~/blog.git
```

The original repo can be deleted. Next, 

```
$ cd ~/blog.git/hooks
$ nvim post-receive
```

Add the following to it:
```
#!/bin/bash

GIT_REPO=$HOME/blog.git
WORKING_DIRECTORY=$HOME/blog-working
PUBLIC_WWW=$HOME/public_html
BACKUP_WWW=$HOME/backup_html
MY_DOMAIN=server_domain_or_IP

set -e

rm -rf $WORKING_DIRECTORY
rsync -aqz $PUBLIC_WWW/ $BACKUP_WWW
trap "echo 'A problem occurred.  Reverting to backup.'; rsync -aqz --del $BACKUP_WWW/ $PUBLIC_WWW; rm -rf $WORKING_DIRECTORY" EXIT

git clone $GIT_REPO $WORKING_DIRECTORY
rm -rf $PUBLIC_WWW/*
/usr/bin/hugo -s $WORKING_DIRECTORY -d $PUBLIC_WWW -b "https://${MY_DOMAIN}"
rm -rf $WORKING_DIRECTORY
trap - EXIT
```
> Note: change the `server_domain_or_IP` to the public IP of the website.

Here's what we do here:
Every time the git repo is pushed from the development server to the production server (i.e this server), the repo is automatically pulled <!-- TODO: from where? --> and build with hugo and automatically copied to ~/public_html. 

`set -e` makes the program quit immediately in case of errors. The trap command does the clean up when this happens. 

Set up git remote in the development sever: 

```
$ git remote add prod username@production_domain_or_IP:my-website.git
```

When a change is made in the devserver just push to the prodserver with `git push prod` and it should work!