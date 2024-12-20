---
layout: post
title: Exploring Crystal language
excerpt_separator: <!--more-->
---

These days I decided to explore the Crystal programming language, a high-performance, statically-typed programming language with Ruby-inspired syntax. To do so, I decided to port NativeDump and TrickDump to it.

<!--more-->


Repositories: 

- [NativeDump](https://github.com/ricardojoserf/NativeDump/tree/crystal-flavour)

- [TrickDump](https://github.com/ricardojoserf/TrickDump/tree/crystal-flavour)

<br>

-------------------------------------------

# Motivation

Crystal can execute Windows API calls, including NTAPI calls, by leveraging its C bindings and low-level system capabilities.

Crystal provides the lib keyword, which allows defining bindings to shared libraries or system APIs written in C. This includes the Windows API and NTAPI.

Using lib declarations, you can map the Windows API functions from kernel32.dll, user32.dll, or NTAPI functions from ntdll.dll into your Crystal code. 

The @[Link] annotation is typically required when you want to explicitly link with a specific library that is not automatically linked by the system. For example, when calling functions from ntdll.dll or other custom shared libraries.

This is an example for defining the NtQueryInformationProcess NTAPI function:

```
@[Link("ntdll")]
lib Ntdll
  fun NtQueryInformationProcess(
    process_handle : Void*,
    process_information_class : UInt32,
    process_information : Void*,
    process_information_length : UInt32,
    return_length : Pointer(UInt32)
  ) : Int32
end
```


Then you can call it, for example to get the Info Class PROCESS_BASIC_INFORMATION (value 0) from the current process (value -1): 

```
status = Ntdll::NtQueryInformationProcess(Pointer(Void).new(-1), 0, Pointer(Void).null, 0, Pointer(UInt32).null)
puts "Status: #{status}"
```

Knowing this, I created CrystalDump, a NativeDump port entirely written in Crystal; and a port of TrickDump to Crystal as well.

Exploring Crystal's potential was inspired by [Rastamouse](https://twitter.com/_rastamouse)'s blog post "Crystal Malware" from earlier this year: [Crystal Malware](https://rastamouse.me/crystal-malware/).

<br>

-------------------------------------------

# CrystalDump

CrystalDump is a port of NativeDump written in Crystal lang, designed to dump the lsass process using only NTAPI functions:

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
crystaldump.exe [-o OUTPUTFILE ] [-r]
```

- **Output file** (-o, optional): Dump file name

- **Remap ntdll** (-r, optional): Remap the ntdll.dll library


<br>

By default it creates a file named "crystal.dmp":

```
crystaldump.exe
```

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/crystal_1.png)


Using the parameter *-r* it remaps the ntdll.dll library:

```
crystaldump.exe -r
```

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/crystal_2.png)


The parameter *-o* is used to change the output file name:

```
crystaldump.exe -r -o document.pptx
```

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/refs/heads/master/images/nativedump/crystal_3.png)

<br>

------------------

## Build

To build the binary, use a command like:

```
crystal build crystaldump.cr --release
```

<br>

-------------------------------------------


# TrickDump - Crystal port

This branch implements the same functionality as the main branch using Crystal lang.

```
lock.exe  [-j JSON_NAME ] [-r]
```
```
shock.exe [-j JSON_NAME ] [-r]
```
```
barrel.exe [-j JSON_NAME ] [-z ZIP_NAME] [-r]
```

Options:

- **JSON file name** (-j, optional): JSON file name

- **ZIP file name** (-z, optional): ZIP file name

- **Remap ntdll** (-r, optional): Remap the ntdll.dll library

<br>

By default the programs do not remap the ntdll.dll and create the files "lock.json", "shock.json",  "barrel.json" and "barrel.zip":

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/crystal_trick1.png)


It is possible to remap the ntdll.dll library using the parameter *-r* and change the default file names with *-j* and *-z*:

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/crystal_trick2.png)


Then use the *create_dump.py* script to generate the Minidump file in the attack system:

```
python3 create_dump.py [-l LOCK_JSON] [-s SHOCK_JSON] [-b BARREL_JSON] [-z BARREL_ZIP] [-o OUTPUT_FILE]
```

![img3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/crystal_trick3.png)

<br>

-------------------------

## Trick: All in one

If you prefer to execute only one binary, Trick.exe generates a ZIP file containing the 3 JSON files and the ZIP file with the memory regions:

```
trick.exe [-z ZIPNAME] [-r]
```


It creates the ZIP file locally, optionally using a ntdll.dll overwrite method:

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/crystal_trick4.png)


With a ZIP file like this, you can unzip it and create the Minidump file using *create_dump.py* later:

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/crystal_trick5.png)

<br>

-------------------------

## Build

First resolve and install dependencies ([crystal-garage/crystal-zip64](https://github.com/crystal-garage/crystal-zip64) is the only one):

```
shards install
```

Then build the binaries you need:

```
crystal build lock.cr --release
```
```
crystal build shock.cr --release
```
```
crystal build barrel.cr --release
```
```
crystal build trick.cr --release
```

<br>