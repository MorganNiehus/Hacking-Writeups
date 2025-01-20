
Released July 2nd, 2024
Difficulty Intermediate (community rated hard)

Hey all! today I am going to demonstrate the compromise of BackupBuddy hosted by the Offsec Proving grounds.  BackupBuddy started with a simple php file manager that was subject to default creds. The website was vulnerable to a directory traversal attack that lead to an exposed SSH key and a user shell. A vulnerable SUID binary led to a Shared Library misconfiguration which granted me a  root shell.

To begin I start with my go to nmap scan that will enumerate services and versions on all ports at a faster rate. Ill also save my results for later viewing.

nmap -sC -sV -p- --min-rate 10000 192.168.188.43

![[Proving Grounds/BackupBuddy/nmap.PNG]]

Judging from the output this is clearly a Ubuntu linux machine with ssh on 22 and an Apache web server on port 80. The best place for me to start is by checking out the webserver.

![[loginNeeded.PNG]]

It's a very simple login interface with a link to PHP File Manager. Clicking on the link takes you to the Github hosting the code. This appears to be the backbone of Tiny PHP File Manager.

![[GithubPHP.PNG]]


First thing I notice is that the code is extemely old. This might work in my favor. Checking on the issues and Security tabs, hoping for low hanging fruit, I find nothing interesting.

![[issues.PNG]]

![[securityAdvise.PNG]]

Looking back at the README.md I do notice that this application is shipped with default credentials. These are worth a try!

![[Proving Grounds/BackupBuddy/defaultCreds.PNG]]

Ill go ahead and take note of those creds and return to the login prompt and input the default creds.

![[added creds.PNG]]

To my surprise, they worked like a charm!

![[logged_in.PNG]]

Obviously this site is running PHP, and because this is some sort of file manager I wanted to see if there was somewhere to upload a php script for a command injection. Unfortunately, There isn't.

Clicking into backup, all I see is a stock photo. You can find the exact photo if you put the name of it into google.

![[stockPhoto.PNG]]

The same photo is in the "important_images" directory. Taking note that the owner is Brian for both images.

![[important_images.PNG]]

Clicking on the image does show the entire path of where the file is located on the webserver.

![[location.PNG]]

While I was clicking around. I noticed that every destination is navigated to by the "p" parameter in a directory traversing format with the unicode character for "/" being %2F.

![[pparameter.PNG]]

I tried to test for a directory traversal attack by grabbing /etc/passwd by inputting "p=backup/../../../../etc/passwd", but unfortunately it didn't work and the parameter became blank.

![[failed-etc-passwd.PNG]]

So what if I tried moving back a directory by inputting "backup/../../" or even simpler "../../"?

![[directoryTraversal.PNG]]

Aha! I managed to escape the limits of backup. With this power, I can view any file on the machine that's available to Brian. I can take this all the way to "/" 

![[backtoroot.PNG]]

I immediately search for ways I can get a shell. My best bet is that Brian has an ssh key in his home folder so Ill check there first.

![[found-ssh.PNG]]

Jackpot!

![[Proving Grounds/BackupBuddy/idrsa.PNG]]

I see now that my capture of /etc/passwd would have worked. I just needed to do "//" before /etc. No worries!

Now that I have the id_rsa for the user Brian I need to see if I can log in as Brian via SSH.

Ill start by downloading the key onto my local machine.

![[idsaved.PNG]]

Then I will go ahead and run ssh2john so I can crack the password.

ssh2john id_rsa > id_rsa.hash

![[ssh2john.PNG]]

john id_rsa.hash --wordlist=/usr/share/wordlist/rockyou.txt

I already have it cracked, but the password is "eugene", easy day!

![[hashCrack.PNG]]

Now I will log in via ssh

ssh -i id_rsa brian@192.168.188.43

![[loged-in-ssh.PNG]]

Go ahead and run bash, makes the shell much better!

/bin/bash

![[binbash.PNG]]


By muscle memory I will run sudo -l to check for an easy privilege escalation.

sudo -l

![[nojoyonsudo.PNG]]

Unfortunately, I don't have the sudo password for Brian.

Before I run linpeas, I just want to take a small look around. I noticed there is a custom binary located in /opt and even better, it's and SUID binary!

![[sudi.PNG]]

This means that it will always execute as the root user. I need to find a way to take advantage of this. Ill go ahead and run it to see what the behavior is.

./backup

![[failedbackup.PNG]]

It immediately fails. I tried a few combinations and inputs and it always ended up with the failure. Because this is a binary I cant read the code, but I can run strings to see if there is anything hidden behind the scenes.

strings backup

![[interesting.PNG]]

It looks like the program is relying on a shared library located in /home/brian/.config. Ill go ahead and check that out.

![[doesntExist.PNG]]

Looks like the shared library is missing. That explains why it is failing. Because this location is writable by me and specifically being called by the application, I can take advantage of this SUID and write my own libm.so that executes my own code as the root user. Gurkirat Singh (https://tbhaxor.com/exploiting-shared-library-misconfigurations/) has a great tutorial and explanation on how to accomplish this. I will use his script with a slight modification.

````c
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void bad_stuff() {
        setuid(0);
        setgid(0);
        system("/bin/sh -i");
}

````

Very important to set the setuid and setgid or else it will not work.

Ill go ahead and create the directory and add my C file.

cd /home/brian

mkdir .config

cd /.config

vi ma_libm.c

Its very important to compile the script on the victim machine, Ill do that by running

gcc -shared -fPIC -o libm.so ma_libm.c

![[madeCscript.PNG]]

All I need to do now is run the binary!

![[root!.PNG]]

And grab the proof.txt file!

![[Proving Grounds/BackupBuddy/rootflag.PNG]]

Thank you for taking the time to read this write up! Happy Hacking!




