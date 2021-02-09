---
title: "Guide to setup Hugo website with Nginx in FreeBSD"
date: 2021-02-09T18:46:13+05:30
---


# Introduction
> Note: There is no particular reason I chose to use Hugo, Nginx and FreeBSD.

> Note 2: Most of this is a rip-off of the excellent documentation at DigitalOcean. Its compiled here for convenience.

Before starting setup SSH.

# First things

Update the system:

`sudo freebsd-update fetch install`

Install the following:
* nvim
* bash
* hugo
* nginx

Change the default shell to bash:

`$ sudo chsh -s bash freebsd`

## Setting up firewall

Add the following to /etc/rc.conf

`
firewall_enable="YES"
firewall_quiet="YES"
firewall_type="workstation"
firewall_myservices="22 80 443"
firewall_allowservices="any"
firewall_logdeny="YES"
`

The third line opens the ports 22, 80 and 443 for SSH, HTTP and HTTPS respectively.

To start the firewall:

`$ sudo service ipfw start`

To configure how many denials per IP to log, append the following to /etc/sysctl.conf:

`net.inet.ip.fw.verbose_limit=5`

## Setting up the timezone

FreeBSD provides a TUI for this purpose, which is quite nice. To access it run:

`$ sudo tzsetup`

You can now enable NTP. To prevent NTP from failing during boot, its recommended to sync it on start. For this run the following:

`sudo sysrc ntpd_enable="YES"
$ sudo sysrc ntpd_enable="YES"`

Also don't forget to start the service:

`$ sudo service ntpd start`


