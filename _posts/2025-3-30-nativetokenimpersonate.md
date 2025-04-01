---
layout: post
title: NativeTokenImpersonate - Token Impersonation using only NTAPIs
excerpt_separator: <!--more-->
---

Impersonate users by stealing tokens to create new processes using only NTAPI functions. It supports two types of impersonation, one similar to CreateProcessWithToken and the other to ImpersonateLoggedOnUser.


<!--more-->

It also enumerates process information and includes the capability to remap the ntdll.dll library creating a suspended process, again using only NTAPI functions.

Repository: [https://github.com/ricardojoserf/NativeTokenImpersonate](https://github.com/ricardojoserf/NativeTokenImpersonate)

<br>

The high-level overview for stealing tokens creating a new process (like [CreateProcessWithToken](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createprocesswithtokenw)):

![esquema](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/esquema.png)

For impersonating a user without a new process, altering the main thread of the parent process (like [ImpersonateLoggedOnUser](https://learn.microsoft.com/en-us/windows/win32/api/securitybaseapi/nf-securitybaseapi-impersonateloggedonuser)):

![esquema2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/esquema2.png)

And for listing processes or SIDs:

![esquema3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/esquema3.png)


It uses other NTAPI functions for other tasks:

- `NtOpenProcessToken` and `NtAdjustPrivilegesToken`: Enable privileges.

- `RtlInitUnicodeString`, `RtlUnicodeStringToAnsiString` and `RtlConvertSidToUnicodeString` : Manage strings.

- `RtlAllocateHeap` and `RtlFreeHeap`: Manage heap memory.

- `RtlCreateProcessParametersEx` and `RtlDestroyProcessParameters`: Manage Process Parameters.

- `NtClose`: Close object handles.


Please note that:

- It seems it is not possible to translate the SID to a username with ntdll.dll functions, but it can be quickly translated using wmic or Powershell.

- The user needs to hold the *SeAssignPrimaryTokenPrivilege* privilege to steal a token and create a new process, check the "Common errors" section for more details. For the second type of impersonation it is enough with *SeDebugPrivilege* and *SeImpersonatePrivilege*.

- To eliminate the need for `LookupPrivilegeValue` from Kernel32.dll, the LUID values of *SeAssignPrimaryTokenPrivilege*, *SeDebugPrivilege* and *SeImpersonatePrivilege* are hardcoded as macros at the beginning of the file.

- The program only uses `LoadLibraryA("ntdll")` and `GetProcAddress(hNtdll, "NtReadVirtualMemory")` from Kernel32.dll to initialize the rest of the function addresses. I guess you can hardcode this one :)

- It is probably better to use indirect syscalls instead of remapping ntdll, but it was more simple for the PoC.

<br>

--------------------------

## Usage

### Steal token (creating a new process)

```
NativeTokenImpersonate.exe -pid <PID> [-bin <BINARY>] [-args <ARGUMENTS>] [-remap] [-system]
```

- *-pid*: PID of the process to impersonate.
 
- *-bin*: Binary path to run (default: "C:\Windows\System32\cmd.exe").

- *-args*: Arguments for the binary (default: None).

- *-remap*: Remap the ntdll.dll library (default: False).

- *-system*: Skip enabling privileges if already running as SYSTEM (default: False).

<br>

**Example** - Impersonate the user owning the process with PID 16172 and open a Cmd shell:

```
NativeTokenImpersonate.exe -pid 16172
```

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_1.png)


**Example** - Impersonate the user owning the process with PID 16172 and run a Powershell command:

```
NativeTokenImpersonate.exe -pid 16172 -bin c:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -args "whoami; pause"
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_2.png)


**Example** - Remap the ntdll.dll library, impersonate the user owning the process with PID 16172 and open a Powershell shell:

```
NativeTokenImpersonate.exe -pid 16172 -remap -bin c:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
```

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_3.png)


<br>

---------------------------

### Steal token (using the same thread)

```
NativeTokenImpersonate.exe -thread -pid <PID> [-remap] [-system] [-rev2self]
```

- *-thread*: Impersonate at thread level. If not used, it will spawn a new process.

- *-pid*: PID of the process to impersonate.
 
- *-remap*: Remap the ntdll.dll library (default: False).

- *-system*: Skip enabling privileges if already running as SYSTEM (default: False).

- *-rev2self*: Revert token to its default value, like RevertToSelf (default: False).

<br>

**Example** - Impersonate the user owning the process with PID 5552 at thread level, skipping enabling privileges because SYSTEM already has them. In this case, the results from "whoami" are the same but it is possible to access a file only readable by the low-privileged user:

```
NativeTokenImpersonate.exe -thread -pid 5552 -system
```

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_4.png)

**Example** - After impersonation, if you were not running as SYSTEM you may need to restore the original security context of the thread:

```
NativeTokenImpersonate.exe -rev2self
```

![img4b](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_4b.png)

<br>

---------------------------

### List processes

```
NativeTokenImpersonate.exe -proc [-filter <FILTER>] [-remap]
```


- *-filter*: Filter processes containing this value (default: None).

- *-remap*: Remap the ntdll.dll library (default: False).

<br>

**Example** - Print all processes:

```
NativeTokenImpersonate.exe -proc
```

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_5.png)


**Example** - Remap the ntdll.dll library and print all processes containing the string "winlo":

```
NativeTokenImpersonate.exe -proc -filter winlo -remap
```

![img6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_6.png)


<br>

---------------------------

### List SIDs 

```
NativeTokenImpersonate.exe -sids [-filter <FILTER>] [-remap]
```

- *-filter*: Filter SIDs containing this value (default: None).

- *-remap*: Remap the ntdll.dll library (default: False).

<br>

**Example** - Print all the SIDs:

```
NativeTokenImpersonate.exe -sids
```

![img7](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_7.png)


**Example** - Remap the ntdll.dll library and print the SIDs containing the string "S-1-5-9":

```
NativeTokenImpersonate.exe -sids -filter S-1-5-9 -remap
```

![img8](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_8.png)

<br>

---------------------------

## Common errors

If you are not running as administrator you might find this error:

![img9](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_9.png)

If you are running as administrator but the user does not hold the *SeAssignPrimaryTokenPrivilege*, it is not possible to create a new process with the duplicated token (at least with this tool). You can add the privilege to the user using "secpol.msc" in "Security Settings" > "Local Policies" > "User Rights Assignment" > "Replace a process level token". 

As an alternative, you can try the second type of impersonation (using the *-thread* flag) which requires the more common *SeDebugPrivilege* and *SeImpersonatePrivilege* privileges:

![img10](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativetokenstealer/Screenshot_10.png)

<br>

---------------------------

## References

- [NtCreateUserProcess](https://github.com/capt-meelo/NtCreateUserProcess) by [capt-meelo](https://github.com/capt-meelo) - Used as a reference for implementing NtCreateUserProcess and creating the suspended process.

- [NtCreateUserProcess](https://github.com/BlackOfWorld/NtCreateUserProcess) by [BlackOfWorld](https://github.com/BlackOfWorld) - Used as a reference for implementing NtCreateUserProcess and adding arguments to the call.

- [SharpImpersonation](https://github.com/S3cur3Th1sSh1t/SharpImpersonation) by [S3cur3Th1sSh1t](https://github.com/S3cur3Th1sSh1t) - Used as a reference for thread impersonation.

- [NativeNtdllRemap](https://github.com/ricardojoserf/NativeNtdllRemap) - Template for ntdll.dll remapping using only NTAPI functions.

- [T1134.001 - Token Impersonation/Theft](https://attack.mitre.org/techniques/T1134/001/) by MITRE.

<br>
