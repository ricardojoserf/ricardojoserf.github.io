---
layout: post
title: goNtdllOverwrite - API Unhooking in Golang
excerpt_separator: <!--more-->
---

Overwrite ntdll.dll's ".text" section using a clean version of the DLL using Golang.


<!--more-->

Repository: [https://github.com/ricardojoserf/goNtdllOverwrite](https://github.com/ricardojoserf/goNtdllOverwrite)


It can help to evade security measures that install API hooks such as EDRs. 

The unhooked version of the DLL can be obtained from:

- A DLL file already on disk - For example "C:\Windows\System32\ntdll.dll".
- The KnownDlls folder ("\KnownDlls\ntdll.dll").
- A process created in debug mode - Processes created in suspended or debug mode have a clean ntdll.dll.

<br>

---------------------------------

### Installation

After installing Git, run the following commands:

```
go env -w GO111MODULE=off
go get golang.org/x/sys/windows
```

<br>

---------------------------------

### From disk

Get the clean ntdll.dll from disk. You can specify a file path or use the default value "C:\Windows\System32\ntdll.dll":

```
go run goNtdllOverwrite.go -o disk [-p PATH]
```

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/goNtdllOverwrite/Screenshot_1.png)

<br>


### From KnownDlls folder

Get the clean ntdll.dll from the KnownDlls folder:

```
go run goNtdllOverwrite.go -o knowndlls
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/goNtdllOverwrite/Screenshot_2.png)

<br>


### From a debug process

Get the clean ntdll.dll from a new process created with the DEBUG_PROCESS flag. You can specify a binary to create the process or use the default value "C:\Windows\System32\calc.exe":

```
go run goNtdllOverwrite.go -o debugproc [-p PATH]
```

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/goNtdllOverwrite/Screenshot_3.png)

<br>

-------------------------------

### Links

- .NET implementation: [SharpNtdllOverwrite](https://github.com/ricardojoserf/SharpNtdllOverwrite)

- Python implementation: [pyNtdllOverwrite](https://github.com/ricardojoserf/pyNtdllOverwrite)

<br>
