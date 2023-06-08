# CyberDiscord Open Linux Writeup
## Foreword
I'm a former CyberPatriot IX National Finalist, specializing in Linux, now working as a SysAdmin/Security Engineer. I've been very much out of the loop for many years, but this was an excellent competition and I had a lot of fun! These competitions have come a long way from maxing out Regionals (Semis for all you young folk) images in 3 hours and I love the difficulty increase.

My scores were above average on both Ubuntu and Fedora, but notably they were not in the upper tier of either image scores. I'm making this writeup partly to encourage those teams to make their own write-ups. :)

Also, my team was a two-man squad, and my single teammate doing Windows was one of the exclusive group of successful goose hunters. Maybe I'll bug him to do a write-up on that. 

**Ubuntu Score**: 38 

**Fedora Score**: 11

## Ubuntu
### Forensics
The details of the Forensics questions have since escaped me, so I'll deal in broad strokes here.

#### NGINX Logs Forensics Question
This was the first question I took a look at. I quickly pulled up the logs for NGINX and noticed they were empty. Some brief scouring through SystemD logs leading nowhere, and a pointed hint in the README about a custom database led me to save this question for later. Unfortunately, I never did get back around to it, but I suspect the database holds the answers we seek.

#### Credentials Harvester Question
I very much enjoyed this question. :) 

##### Finding the Credentials
I believe the location of the credentials harvester was called out in the README, but even if not, one of the first things you should do is check served files, so this part was easy. As for finding the other end, the credential harvester itself... there's a funny quirk whenever we used `sudo` on this machine, a very interesting message indeed. I noticed the fun sudoers message immediately, and `Ctrl-C`'d out of it, not wanting to give my password away to a credentials harvester. 

Interestingly enough, I still gained root access despite canceling out of both the hijacked sudoers prompt and the actual sudoers prompt. File away insecure authentication methods for later.

Connecting the two pieces of the puzzle, whenever a password was entered into the hijacked prompt, that password would find itself in the FTP served credentials file. 

##### Locating the Harvester
With the full pipeline of the script found, my next step seemed obvious: search for the processes that have most recently edited the credentials file. 

I immediately tried `lsof`, however this only gets the process *currently* accessing a file, not previous edits. I tried looking through `crontab`, and found a lovely backdoor to remove, but no progress on the harvester. Some googling led me to [this](https://unix.stackexchange.com/questions/186539/how-to-audit-access-to-any-file-or-folder-within-a-given-path-for-specific-user) StackExchange answer, which led me down the path of `auditctl`. 

A successful implementation of `auditctl` and triggering the harvester gave me all the information I needed: *bash* was updating the credentials. I spent **far too long** thinking this was an error in my auditing, as I expected more information. What I wanted was a script location, and it unfortunately took an hour for my brain to realize that bash has several built-in script locations I could check. 

Once I recovered from my smooth brain experience, I immediately set about checking the `.bashrc` of my users. I had a quick dive into the `$PATH`, looking for any obvious `CREDENTIAL-HARVESTER-SCRIPT` type executables, but without any luck. However, there is a master `.bashrc.`, upon which all `.bashrc`'s inherit from, and upon checking that we found our good friend the credentials harvester. He didn't live much longer. :)

#### IRC Server Question
##### IRC Server
For finding active services, I'm a very big fan of trusty old `nmap localhost` for its nice clean interface - just remember to remove it afterwards if you find its pre-installed. Finding the IRC server was very straightforward, and while I was here I also shutdown 2 other unauthorized services.

The question asks where the script that prevents the removal of the server is located - so what better way to get that out of hiding than to try and remove the IRC server? 

##### IRC Server Removal
###### The Part Where I Screwed Up
Sure enough, `apt` locked up and refused to budge (I closed that terminal/process), but it also presented the way forward: find the process that hijacked the `dpkg` lock system. I could have tackled it in the same way as the previous question, with `auditctl`, but I was disappointed in the vague output of that program and did some more googling. 

I stumbled upon an [answer](https://unix.stackexchange.com/questions/163603/how-to-find-source-of-spawning-process) that pointed me to `sysdig`. Great, I have a tool that seemed like it would fit the job perfectly. A simple `apt install sysdig` and... `apt` is still completely locked up, I can't install it. Fantastic.

###### Fixing my Screw Up
`apt` is still locked up because it has marked the IRC server for removal, and it didn't successfully complete the removal last time - so it will try again. In the meantime, `dpkg` is locked and I cannot kill the process. Attempting a quick fix, I got the process number of the lock and ran something like `kill -9 $PROC & apt install sysdig`, hoping that the asynchronous startup of `apt` would get out in front of whatever was locking `apt` up. This worked for starting a new install process, but even when called to install something, it first needed to complete its queued removal... which started the whole process all over again.

The actual fix is to cancel the queued deletion, with an `apt install --reinstall insipircd`. With my package manager back under my control, I proceed to install `sysdig`.

##### Revenge 
Setting up `sysdig` to monitor the `dpkg` lock file, it gave me much more useful output than `auditctl` ever did. I tried the removal again and it immediately pinpointed the source of the script as a `.postrm` file, which is a script that tells the package manager what to do to cleanup the removed package.

Looking in our IRC server's `.postrm` file, at the very end of it was a hook to reinstall the package in the middle of removing it, thus completely hijacking `apt`.

This, everyone, is why Arch begs you to inspect the installation of everything you install from the AUR. 

### SUDOERS
At this point, I was finally ready to run my script, which gave me plenty of points for removing unauthorized users, password policies & configuration, removing hacking programs, firewalls, and sysctl config. Once that was done, I configured & started updates in the background and moved back to the issue I had found before: insecure sudo config.

I took a quick overview of `/etc/sudoers` and nothing seemed out of place to me. I then took a peek into `/etc/sudoers.d/` and there I found a very suspicious file. Everything in this directory is considered in the sudoers configuration by default (even the README!), so an extra file here almost certainly needs to go. When opened, however, it was very big and very blank. I had never encountered this particular phenomenon before - a completely blank file, but with *plenty* of empty lines and a non-zero file size. 

With my trusty first and last option, googling, there's a very promising [lead](https://unix.stackexchange.com/questions/505558/cat-shows-nothing), and sure enough, there's an insecure configuration hidden in that file. Deleted.

---
At this point, I had spent about 3 hours on Ubuntu, and my plan now was to spend the last hour on Fedora getting the easy points, to maximize my score. I didn't do much service configuration, so I suspect I missed a large amount of points there, and I also skipped doing two forensics questions. 

Overall, I really really enjoyed this image and I wish I had more time with it!

## Fedora
I took a brief look at the Forensics Questions, to see if I could knock any out quickly. Since the Ubuntu forensics took me far too long, I didn't have much hope, but when I saw that regex question hope sparked in me and I immediately went to work. What followed was a humbling experience, as after 15 minutes I couldn't get either `grep` or `ripgrep` to accept that expression as valid. I need to work on my regex, it seems.

### Minecraft
I saw Minecraft installed on the desktop as soon as I booted up the image and I thought I would be removing it shortly. I was so excited that it was necessary to keep and was even points to configure - the single most fun idea for points I've ever seen, kudos to the maker of this image.

The first step in installing Minecraft is to run the `.jar` installation file, however it wasn't compatible with the version of Java pre-installed on the image. A quick update to the latest `open-jdk` version and I was able to install it, no problem. 

Having never configured a Minecraft server before, I went into the configuration file line-by-line and configured anything that seemed relevant. Most of the options were straightforward, but I was unsure about the whitelist in particular, so I found a [guide](https://help.akliz.net/docs/setup-a-whitelist-for-your-minecraft-server) that told me to go into the console and enable the whitelist manually, so I did. 

At this point, I tried to launch Minecraft, but found that it wouldn't run. Very sad. Even sadder, I believe running the Minecraft jar reset my `server.properties` config file, so I had to reconfigure that to get my Minecraft points back. A few more attempts at launching Minecraft didn't work, so sadly I didn't get to spend my last 30 minutes punching trees.

### Updates
With my final minutes, I had enough time to run updates on Fedora. This gave me several non-update related points, which I was happy to take for free. I took some time to quickly run through `/etc/passwd` for anything suspicious (`/bin/sync` with a user group of `0` looked mighty suspicious, but I got no points for it and I suspected it wasn't as suspicious as it looked with my 2 minutes of googling) but in the end, I didn't get any more points before time ran out.

---
I got to configure a Minecraft server and also got free points for updates, 10/10 image.