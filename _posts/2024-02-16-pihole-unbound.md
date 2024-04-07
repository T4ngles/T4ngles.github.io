---
layout: post
title: pihole + unbound
tags: pihole unbound dns resolver ads block
categories: architecture
---

_Adding the popular Unbound resolver to my pihole DNS sinkhole_

Following adding a pihole to filter out ads and also suss domains (I'm looking at you robovac!) I decided to also add unbound, a recursive DNS resolver to ensure that all DNS resolution queries are not passed to another external DNS provider. The guide [(https://docs.pi-hole.net/guides/dns/unbound/)](https://docs.pi-hole.net/guides/dns/unbound/) provided on the official pi-hole docs website provides a very good explanation of the how pi-hole functions, why adding a recursive DNS resolver is good if you **trust no one** and also runs you through all the steps to install unbound.

I won't go over the steps for installation here as the guide is very good and I recommend anyone follow it. I will go over some issues I encountered and things I changed.

## Root Hints

For the unbound service to work it will need to be able to start the DNS recursive query from the root DNS servers so it needs to have an idea where those are located. This is accomplished through the root hints file, located at `/var/lib/unbound/root.hints`, which gives the ip addresses of the root name servers. It's recommended to update this about once every 6 months since it changes infrequently and since I can be forgetful I added a cron job for it.

`0 0 1 */6 * sudo wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints`

## Enabling Unbound Logging

To enable logging I specified the log location in the `/etc/unbound/unbound.conf.d/pi-hole.conf` file by just uncommenting out the default logfile: line and adding log-queries.

```
server:
# If no logfile is specified, syslog is used
    logfile: "/var/log/unbound/unbound.log"
    log-time-ascii: yes
    verbosity: 4
    log-queries: yes
```

After setting these configs I restarted the service and then ran dig commands both specifying the local unbound service and a normal one and checked the logs to make sure it was working and at the right verbosity level.

```
sudo service unbound restart
dig virustotal.com @127.0.0.1 -p 5335
dig exploit-db.com
tail /var/log/unbound/unbound.log
```
However the verbosity level wasn't right it was only one line with basic query info. Turns out I needed to edit the .conf file located at either of the following two places:

`/etc/unbound/unbound.conf` if you are using unbound only locally which in this case is true, the pihole service is sending requests to the unbound service on the same pi through the loopback address 127.0.0.1:5335
`/usr/local/etc/unbound.conf` if you are using the ports version of unbound, that is other clients on the network are querying the unbound service.

You will notice that in these unbound.conf files there is a reference to other .conf files:
`include-toplevel: "/etc/unbound/unbound.conf.d/*.conf"`
So what was happening was all the settings in `pi-hole.conf` which is matched in `*.conf` were loaded first and then those in `/etc/unbound/unbound.conf` overwrote those changing the verbosity level back to the default of 1. So instead of changing the settings in this .conf file I just commented them out and therefore everything is deteremined in the `pi-hole.conf` file.

After setting the logfile location, verbosity level and setting log-queries to yes the logs at  `/var/log/unbound/unbound.log` started giving me the correct verbosity. See `man unbound.conf` for more details on the settings. 


## Enabling Pihole Logging
Pihole should already be generating logs stored in `/var/logs/pihole.log` but you can check the config file at `/etc/pihole/setupVars.conf` to see if it's set to true. This conf file also stores some other settings most notably the web client password which is a double SHA256 hash of the password. You can try doing `echo -n YOURPASS | sha256sum | awk '{printf $1 }' | sha256sum` to check it matches the password you set and then go try to crack it with john hehe.

## Monitoring Logs
Now that we have both logs going we can set up tails on the terminals and watch the logs populate

```
sudo tail -f /var/log/pihole.log 
sudo tail -f /var/log/unbound/unbound.log
```
It looks a bit mundane in black and white though so took the chance to learn some awk magic for colouring and the following command checks for pihole queries if they contain forwarded, SERVFAIL or query and colours them green, red and yellow respectively otherwise leave out the colour.

`sudo tail -f /var/log/pihole.log | awk '{ if ($0 ~ /forwarded/) { print "\033[32m" $0 "\033[0m" } else if ($0 ~ /SERVFAIL/) { print "\033[31m" $0 "\033[0m" } else if ($0 ~ /query/) { print "\033[33m" $0 "\033[0m" } else { print $0 } }'`

![Logs](/assets/piholeunboundlogs.png)

This does take a while to start up though so be patient for the coloured logs. Not too sure how I'd want to colour the unbound queries though, they would more likely be if the domain was of interest which probably isn't something that can be parsed without external help. Maybe VirusTotal can help, maybe next _time_.

