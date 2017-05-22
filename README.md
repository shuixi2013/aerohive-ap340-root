# aerohive-ap340-root
Python script to change the root password on the AeroHive AP340 with HiveOS &lt; 6.1R5
The result is direct root command line access via /bin/sh rather than the locked down AH CLI.

No dependencies required. Just need the IP address of the AP340 and the code will set the root password to "password"

This works by taking advantage of the LFI vulnerability described here:
https://www.exploit-db.com/exploits/34038/

The steps are as follows:

1. Poison the log file in /var/log/messages by injecting PHP code into the username field of the login page
2. Call the uploaded PHP shell with the LFI URL, changing the root password for SSH
3. Login with SSH as root using password "password"
