---
layout: post
title: C# implementation of GetModuleHandle for remote processes
excerpt_separator: <!--more-->
---

GetModuleHandle implementation in C# using only the NTAPIs: NtQueryInformationProcess, NtReadVirtualMemory and (optionally) NtOpenProcess.


<!--more-->

It works like the [GetModuleHandle](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea) WinAPI but for remote processes: it takes a PID and DLL name, walks the PEB structure and returns the remote DLL base address.

This is useful to get the base address of a DLL in the system which is not loaded in your current process but you know is loaded in other process.

It uses the [NtQueryInformationProcess](https://learn.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntqueryinformationprocess) and [NtReadVirtualMemory](http://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FMemory%20Management%2FVirtual%20Memory%2FNtReadVirtualMemory.html) NTAPIs to get the base address and [NtOpenProcess](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-ntopenprocess) to open the process handle. However, NtOpenProcess code is outside the function in case you want to use process handles directly.

Repository: [https://github.com/ricardojoserf/GetModuleHandleRemote](https://github.com/ricardojoserf/GetModuleHandleRemote)


<br>

-------------------------------------------

Usage of the compiled binary:

```
GetModuleHandleRemote.exe PROCESS_PID DLL_NAME.dll
```

For example, to get the base address of gpapi.dll from Process Hacker:

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/getmodulehandleremote/Screenshot_1.png)

<br>