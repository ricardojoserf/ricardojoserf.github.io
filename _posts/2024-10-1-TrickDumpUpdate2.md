---
layout: post
title: TrickDump update - BOF File and C/C++ ports
excerpt_separator: <!--more-->
---

Updating TrickDump and creating a BOF File.

<!--more-->


# Trickdump - "bof-flavour" branch

This branch implements the same functionality as the main branch but using BOFs (Beacon Object Files).

You can execute the files using Cobalt Strike, TrustedSec's [COFFLoader](https://github.com/trustedsec/COFFLoader) or Meterpreter's [bofloader module](https://docs.metasploit.com/docs/using-metasploit/advanced/meterpreter/meterpreter-executebof-command.html).


-----------------------------------------

## Cobalt Strike

First import the aggressor scripts:

![bof1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF1.png)

Each BOF file has one optional input argument for overwriting ntdll.dll:
- "disk": Using a DLL already on disk. The default path is "C:\Windows\System32\ntdll.dll".    
- "knowndlls": Using the KnownDlls folder.
- "debugproc": Using a process created in debug mode. The default process is "c:\windows\system32\calc.exe".

```
lock <OVERWRITE_TECHNIQUE>
```

![bof2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF2.png)

```
shock <OVERWRITE_TECHNIQUE>
```

![bof3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF3.png)

```
barrel <OVERWRITE_TECHNIQUE>
``` 

![bof4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF4.png)

If you prefer to generate all the files at the same time, run the Trick BOF:

```
trick <OVERWRITE_TECHNIQUE>
```

![bof6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF6.png)

Finally generate the Minidump file:

```
python3 create_dump.py [-l LOCK_JSON] [-s SHOCK_JSON] [-b BARREL_JSON] [-z BARREL_ZIP] [-o OUTPUT_FILE]
```

![bof7](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF7.png)


-----------------------------------------

## COFFLoader

```
COFFLoader64.exe go <BOF_FILE> <OVERWRITE_TECHNIQUE>
```

The argument to overwrite the ntdll library must be generated using COFFLoader's [beacon_generate.py script](https://github.com/trustedsec/COFFLoader/blob/main/beacon_generate.py):
- "disk": Use the value 09000000050000006469736b00
- "knowndlls": Use the value 0e0000000a0000006b6e6f776e646c6c7300
- "debugproc": Use the value 0e0000000a000000646562756770726f6300
  
Examples running each BOF file with a differente overwrite technique:

```
COFFLoader64.exe go lock_bof.o 09000000050000006469736b00
```

![bof8](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF8.png)

```
COFFLoader64.exe go shock_bof.o 0e0000000a0000006b6e6f776e646c6c7300
```

![img9](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF9.png)

```
COFFLoader64.exe go barrel_bof.o 0e0000000a000000646562756770726f6300
```

![img10](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF10.png)

If you prefer to generate all the files at the same time, run the Trick BOF:

```
COFFLoader64.exe go trick_bof.o <OVERWRITE_TECHNIQUE>
```

![img12](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF12.png)


--------------------------------------

## Meterpreter's bofloader module

You can run the BOF files in your Meterpreter session after loading the "bofloader" module and using "--format-string z <technique>" to use a ntdll overwrite technique:

```
load bofloader
execute_bof lock_bof.o --format-string z <OVERWRITE_TECHNIQUE>
execute_bof shock_bof.o --format-string z <OVERWRITE_TECHNIQUE>
execute_bof barrel_bof.o --format-string z <OVERWRITE_TECHNIQUE>
```

![img14](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF14.png)

If you prefer to generate all the files at the same time, run the Trick BOF:

```
execute_bof trick_bof.o --format-string z <OVERWRITE_TECHNIQUE>
```

![img16](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_BOF16.png)


<br>
<br>

--------------------------

# TrickDump - "c-flavour" branch

This branch implements the same functionality as the main branch but using C/C++:

<br>

```
Lock.exe [disk/knowndlls/debugproc]
```
<br>

```
Shock.exe [disk/knowndlls/debugproc]
```
<br>

```
Barrel.exe [disk/knowndlls/debugproc]
```
<br>
You can execute the programs directly without overwriting the ntdll.dll library:

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_C1.png)

Or use one of the three different overwrite techniques ("disk", "knowndlls" or "debugproc"):

![img2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_C2.png)

Then use the *create_dump.py* script to generate the Minidump file in the attack system:

```
python3 create_dump.py [-l LOCK_JSON] [-s SHOCK_JSON] [-b BARREL_JSON] [-z BARREL_ZIP] [-o OUTPUT_FILE]
```

![img4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_C3.png)


-------------------------

## All in one

If you prefer to execute only one binary, Trick.exe generates a ZIP file containing the 3 JSON files and the ZIP file with the memory regions:

```
Trick.exe [disk/knowndlls/debugproc]
```

It creates the ZIP file locally, optionally using a ntdll.dll overwrite method:

![img5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_C4.png)

With a ZIP file like this, unzip it and create the Minidump file using *create_dump.py*:

![img7](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/trickdump/Screenshot_7.png)


<br>
