---
layout: post
title: NimDump - Stealthy LSASS Dumping Using Only NTAPIs in Nim
excerpt_separator: <!--more-->
---

NimDump is a port of NativeDump written in Nim, designed to dump the lsass process using only NTAPI functions.

<!--more-->


NimDump repository: [https://github.com/ricardojoserf/NativeDump/tree/nim-flavour](https://github.com/ricardojoserf/NativeDump/tree/nim-flavour)

NativeDump repository: [https://github.com/ricardojoserf/NativeDump](https://github.com/ricardojoserf/NativeDump)


![esquema](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/crystal_esquema.png)

- NtOpenProcessToken and NtAdjustPrivilegesToken to enable the SeDebugPrivilege privilege
- NtGetNextProcess and NtQueryInformationProcess to get a handle to the lsass process
- RtlGetVersion to get OS information
- NtReadVirtualMemory and NtQueryInformationProcess to get modules information
- NtQueryVirtualMemory and NtQueryInformationProcess to get memory regions information


The tool supports remapping ntdll.dll using a process created in debug mode. For this it uses the NTAPI functions NtQueryInformationProcess, NtReadVirtualMemory, NtProtectVirtualMemory, NtClose, NtTerminateProcess and NtRemoveProcessDebug; and the Kernel32 function CreateProcessW.

<br>

------------------

## Usage

```
nimdump.exe [-r] [-o:OUTPUT_FILE ]
```

- **-r**: Remap the ntdll.dll library

- **-o**: Dump file name

<br>

It creates a file named "n1m.dmp" by default:

```
nimdump.exe
```

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/nim_1.png)

Using the parameter *-r* allows to remap the ntdll.dll library and *-o* to change the output file name:

```
nimdump.exe -r -o:document.docx
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/nim_2.png)

<br>

------------------

## Compilation

In Windows, install the necessary libraries and compile the binary with:

```
nimble install winim zippy
nim c --cpu:amd64 --opt:size --d:release nimdump.nim
```

In Linux, install the compiler, nim and the necessary libraries:

```
sudo apt install mingw-w64
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
source ~/.profile
export PATH=$HOME/.nimble/bin:$PATH
nimble install winim zippy
```

Finally, cross-compile the binary:

```
nim c --cpu:amd64 --opt:size --d:release --os:windows --gcc.exe:x86_64-w64-mingw32-gcc --gcc.linkerexe:x86_64-w64-mingw32-gcc nimdump.nim
```


<br>

------------------

## Notes

- This is a port of NativeDump so it only works if PPL is not enabled

- It works fine on the latest versions of Windows 10 and 11, and has been tested successfully against common AV and EDR solutions.

<br>