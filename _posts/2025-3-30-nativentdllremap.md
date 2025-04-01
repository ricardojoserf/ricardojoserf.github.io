---
layout: post
title: NativeNtdllRemap - Stealthy ntdll.dll Remapping
excerpt_separator: <!--more-->
---

Use only NTAPI functions to stealthily remap ntdll.dll from a clean .text section of a suspended process. This technique bypasses user-mode API monitoring and enhances evasion against security solutions.


<!--more-->

Repository: [https://github.com/ricardojoserf/NativeNtdllRemap](https://github.com/ricardojoserf/NativeNtdllRemap)

<br>

The NTAPI functions used by the tool are:

- **NtCreateUserProcess**: Create a process in suspended state

- **NtProtectVirtualMemory**: Change memory protection

- **NtQueryInformationProcess**: Retrieve process information 

- **NtReadVirtualMemory**: Read memory content

- **NtClose** and **NtTerminateProcess**: Close object handles and processes

- **RtlCreateProcessParametersEx** and **RtlDestroyProcessParameters**: Manage Process Parameters

- **RtlAllocateHeap** and **RtlFreeHeap**: Manage heap memory

- **RtlInitUnicodeString** and **RtlUnicodeStringToAnsiString**: Manage strings

<br>

The program creates the suspended process, calculates the addresses and waits for a key to be pressed:

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_1.png)

Using Process Hacker it is possible to check the initial content of the ntdll.dll's ".text" region in the current process:

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_2.png)

Change the memory contents and click "Write":

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_3.png)

Press any key so the memory is overwritten:

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_4.png)

Click "Re-read" to find the ".text" section has been replaced with the content of the suspended process' ".text" section:

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_5.png)

Finally, press any key so the ".text" section protections are restored and the program finishes: 

![img6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedllremap/Screenshot_6.png)

<br>

**NOTE**: Comment lines 451 and 453 in the *ReplaceNtdllTxtSection* function to prevent the program from pausing until a key is pressed.

<br>