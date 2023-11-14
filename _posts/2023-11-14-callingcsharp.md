---
layout: post
title: Calling C# code from Powershell
excerpt_separator: <!--more-->
---

One of the reasons why I like programming some PoCs using C# is the possibility to later run the code in Powershell. In this post we will see some basic examples and how to prepare your C# code to run it using Powershell.

<!--more-->


Index:

- [Calling C# code from Powershell basic examples](#1)

- [Preparing our custom C# code for Powershell](#2)

- [Final code: getting username with PRTL_USER_PROCESS_PARAMETERS in Powershell](#3)

- [Extra: Getting username with NamedPipe in Powershell](#4)


-----------------------------------------------------

## <a name="1"></a> Calling C# code from Powershell basic examples

A very simple C# example code is using Console.Writeline() to write any message in the console inside a Main() function. As we can see, both the class and the function are public, and the function as no input arguments:

```
$code1 = @"
using System;
namespace TestProject{
	public class Class1 {
		public static void Main()
		{
			Console.WriteLine("Hello World!");
		}
	}	
}
"@
Add-Type $code1
[TestProject.Class1]::Main()
```

The C# code has been defined inside the "code1" variable, and we add the Microsoft .NET class to the PowerShell session using the [Add-Type function](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/add-type?view=powershell-7.3). After adding it, we can call the functions of the class as [Namespace.Class]::Function(Arguments).

If we execute this, the result is the following:

![1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/callingcsharp/Screenshot_1.png)

However, it is important to notice that the function does not need to be called "Main" and it may have input arguments. As you can see in the following example, we are calling the function "RandomName" and now the message in console is sent as an input argument to this function:

```
$code2 = @"
using System;
namespace TestProject{
	public class Class2 {
		public static void RandomName(string message)
		{
			Console.WriteLine(message);
		}
	}
}
"@
Add-Type $code2
[TestProject.Class2]::RandomName("Test text.")
```

If we execute it, the result is very similar to the previous one:

![2](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/callingcsharp/Screenshot_2.png)


It is not even necessary to use a namespace, you can define a class directly and call it, doing this the first example would become:

```
$code1 = @"
using System;
public class Class1 {
	public static void Main()
	{
		Console.WriteLine("Hello World!");
	}
}
"@
Add-Type $code1
[Class1]::Main()
``` 

![6](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/callingcsharp/Screenshot_6.png)

Note that in this case, the nomenclature is [Class]::Function(Arguments).

-----------------------------------------------------

## <a name="2"></a> Preparing our custom C# code for Powershell

We will try to update the code in [this link](https://github.com/ricardojoserf/WhoamiAlternatives/blob/main/PRTL_USER_PROCESS_PARAMETERS/Program.cs) to run it using Powershell. The structure of this code is:

```
using System;
using System.Text;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace PRTL_USER_PROCESS_PARAMETERS
{
    internal class Program
    {
        [DllImport("ntdll.dll", SetLastError = true)] static extern int NtQueryInformationProcess(IntPtr processHandle, int processInformationClass, ref PROCESS_BASIC_INFORMATION pbi, uint processInformationLength, ref uint returnLength);
        ...

        static void GetEnv(int processparameters_offset, int environmentsize_offset, int environment_offset)
        {
        	...
            ReadProcessMemory(hProcess, environment_start, data, data.Length, out _);   
            ...
        }

        static void Main(string[] args)
        {
            ...
        }
    }
}
```


First of all, we can delete the "namespace" declaration and leave the class "Program", as we are adding a .NET class and not a namespace. You do not need to, but I like to do it this way :)

Also, we need to make the class Public or it will not be possible to call it from Powershell.

So the skeleton code will look something like this now:

```
using System;
using System.Text;
using System.Diagnostics;
using System.Runtime.InteropServices;

public class Program
{
    ...
}
```

Then, we find the declaration in .NET of the Main function is like this:

```
static void Main(string[] args) 
```

We will have to remove the "string[] args" part as it generates an error, and we will make the two functions public as in the first example, getting a skeleton code:

```
using System;
using System.Text;
using System.Diagnostics;
using System.Runtime.InteropServices;

public class Program
{
    [DllImport("ntdll.dll", SetLastError = true)] static extern int NtQueryInformationProcess(IntPtr processHandle, int processInformationClass, ref PROCESS_BASIC_INFORMATION pbi, uint processInformationLength, ref uint returnLength);
    ...

    public static void GetEnv(int processparameters_offset, int environmentsize_offset, int environment_offset)
    {
    	...
        ReadProcessMemory(hProcess, environment_start, data, data.Length, out _);   
        ...
    }

    public static void Main()
    {
        ...
    }
}
```

If we try to add this class to Powershell with the Add-Type function, we will still get an error:

![3](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/184a3e9dbdeb2474227f5526c33e1a0daceed5c5/images/callingcsharp/Screenshot_3.png)

The problem are the output variables defined with the underscore character "\_". It is not possible to use this character so we have to define the output variables explicitly (even if we will not use them at all). So a code like this...

```
ReadProcessMemory(Process.GetCurrentProcess().Handle, allocated_address, data, data.Length, out _);
```

... will become this:

```
IntPtr a = IntPtr.Zero;
ReadProcessMemory(Process.GetCurrentProcess().Handle, allocated_address, data, data.Length, out a);
```

-----------------------------------------------------

## <a name="3"></a> Final code: getting username with PRTL_USER_PROCESS_PARAMETERS in Powershell

Finally, we get no errors and can execute the code like this:

```
$code3 = @"
using System;
using System.Text;
using System.Diagnostics;
using System.Runtime.InteropServices;
public class PrtlUserProcessParameters
{
    [DllImport("ntdll.dll", SetLastError = true)] static extern int NtQueryInformationProcess(IntPtr processHandle, int processInformationClass, ref PROCESS_BASIC_INFORMATION pbi, uint processInformationLength, ref uint returnLength);
    [DllImport("kernel32.dll", SetLastError = true)] static extern bool ReadProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, [Out] byte[] lpBuffer, int dwSize, out IntPtr lpNumberOfBytesRead);
    private struct PROCESS_BASIC_INFORMATION { public uint ExitStatus; public IntPtr PebBaseAddress; public UIntPtr AffinityMask; public int BasePriority; public UIntPtr UniqueProcessId; public UIntPtr InheritedFromUniqueProcessId; }
    public static void GetEnv(int processparameters_offset, int environmentsize_offset, int environment_offset)
    {
        IntPtr hProcess = Process.GetCurrentProcess().Handle;
        PROCESS_BASIC_INFORMATION pbi = new PROCESS_BASIC_INFORMATION();
        uint temp = 0;
        NtQueryInformationProcess(hProcess, 0x0, ref pbi, (uint)(IntPtr.Size * 6), ref temp);
        IntPtr PebBaseAddress = (IntPtr)(pbi.PebBaseAddress);
        Console.WriteLine("[+] PEB base:                  \t0x{0}", PebBaseAddress.ToString("X"));

        IntPtr processparameters_pointer = (IntPtr)(pbi.PebBaseAddress + processparameters_offset);
        IntPtr processparameters = Marshal.ReadIntPtr(processparameters_pointer);
        Console.WriteLine("[+] ProcessParameters Address: \t0x{0}", processparameters.ToString("X"));

        // Reference: https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/pebteb/rtl_user_process_parameters.htm
        IntPtr environment_size_pointer = processparameters + environmentsize_offset;
        IntPtr environment_size = Marshal.ReadIntPtr(environment_size_pointer);
        Console.WriteLine("[+] Environment Size:          \t{0}", environment_size);

        IntPtr environment_pointer = processparameters + environment_offset;
        IntPtr environment_start = Marshal.ReadIntPtr(environment_pointer);
        Console.WriteLine("[+] Environment Address:       \t0x{0}", environment_start.ToString("X"));
        IntPtr environment_end = environment_start + (int)environment_size;

        Console.WriteLine("[+] Result:");
        byte[] data = new byte[(int)environment_size];
        IntPtr a = IntPtr.Zero;
        ReadProcessMemory(hProcess, environment_start, data, data.Length, out a);   
        String environment_vars = Encoding.Unicode.GetString(data);
        int found = environment_vars.IndexOf("USERNAME=");
        String rest_String = environment_vars.Substring(found);
        int found2 = rest_String.IndexOf("=");
        int found3 = rest_String.IndexOf("\x00");
        found3 -= found2;
        rest_String = rest_String.Substring(found2 + 1, found3 - 1);
        Console.WriteLine(rest_String);
    }
    public static void Main()
    {
        if (Environment.Is64BitProcess)
        {
            Console.WriteLine("[+] 64 bits process");
            GetEnv(0x20, 0x3F0, 0x80);
        }
        else
        {
            Console.WriteLine("[+] 32 bits process");
            GetEnv(0x10, 0x0290, 0x48);
        }
    }
}
"@
Add-Type $code3
[PrtlUserProcessParameters]::Main()
```

![4](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/184a3e9dbdeb2474227f5526c33e1a0daceed5c5/images/callingcsharp/Screenshot_4.png)

-----------------------------------------------------

## <a name="4"></a> Extra: Getting username with NamedPipe in Powershell

Now that we know how to do it, we can "prepare" and execute any C# code. In the following code snippet, you can see the result of executing the code in [this link](https://github.com/ricardojoserf/WhoamiAlternatives/blob/main/NamedPipe/Program.cs), which also gets the current username:

```
$code4 = @"
using System;
using System.Text;
using System.Threading;
using System.Diagnostics;
using System.Runtime.InteropServices;
public class NamedPipe
{
    [DllImport("kernel32.dll", SetLastError = true)] static extern IntPtr CreateNamedPipe(string lpName, uint dwOpenMode, uint dwPipeMode, uint nMaxInstances, uint nOutBufferSize, uint nInBufferSize, uint nDefaultTimeOut, IntPtr lpSecurityAttributes);
    [DllImport("kernel32.dll", CharSet = CharSet.Ansi)] public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, System.Threading.ThreadStart lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
    [DllImport("advapi32.dll")] public static extern IntPtr NpGetUserName(IntPtr hNamedPipe, IntPtr Username, uint UsernameLength);
    [DllImport("kernel32.dll")] static extern bool ConnectNamedPipe(IntPtr hNamedPipe, IntPtr lpOverlapped);
    [DllImport("kernel32.dll", SetLastError = true)] static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);
    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Auto)] public static extern IntPtr CreateFile(string lpFileName, uint dwDesiredAccess, uint dwShareMode, IntPtr lpSecurityAttributes, uint dwCreationDisposition, uint dwFlagsAndAttributes, IntPtr hTemplateFile);
    [DllImport("kernel32.dll", SetLastError = true)] static extern int WriteFile(IntPtr handle, IntPtr buffer, int numBytesToWrite, out uint numBytesWritten, Boolean lpOverlapped);
    [DllImport("kernel32.dll", SetLastError = true)] static extern int ReadFile(IntPtr hFile, IntPtr lpBuffer, uint nNumberOfBytesToRead, out uint lpNumberOfBytesRead, IntPtr lpOverlapped);
    [DllImport("kernel32.dll", SetLastError = true)] static extern bool CloseHandle(IntPtr hObject);
    [DllImport("kernel32.dll", CallingConvention = CallingConvention.Winapi, SetLastError = true)] static extern IntPtr GetProcessHeap();
    [DllImport("ntdll.dll", SetLastError = true)] static extern IntPtr RtlAllocateHeap(IntPtr HeapHandle, int Flags, int Size);
    [DllImport("kernel32.dll", SetLastError = true)] static extern bool ReadProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, [Out] byte[] lpBuffer, int dwSize, out IntPtr lpNumberOfBytesRead);
    [DllImport("kernel32.dll", SetLastError = true)] static extern bool DisconnectNamedPipe(IntPtr hNamedPipe);

    static int HEAP_ZERO_MEMORY = 0x00000008;
    static uint PIPE_ACCESS_DUPLEX = 0x00000003;
    static uint PIPE_TYPE_MESSAGE = 0x00000004;
    static uint PIPE_READMODE_MESSAGE = 0x00000002;
    static uint PIPE_WAIT = 0x00000000;
    static uint PIPE_UNLIMITED_INSTANCES = 255;
    static uint GENERIC_READ = 0x80000000;
    static uint GENERIC_WRITE = 0x40000000;
    static uint OPEN_EXISTING = 3;
    static int test_string_size = 5;
    static uint MAX_USERNAME_LENGTH = 256;
    static string pipe_name = "\\\\.\\pipe\\LOCAL\\usernamepipe";


    public static void writeFile()
    {
        // Open handle to named pipe
        IntPtr hPipe = CreateFile(pipe_name, GENERIC_READ | GENERIC_WRITE, 0, IntPtr.Zero, OPEN_EXISTING, 0, IntPtr.Zero);
        Console.WriteLine("[+] Handle (CreateFile): \t0x{0}", hPipe);
        if (hPipe == IntPtr.Zero)
        {
            Console.WriteLine("[-] Failure calling CreateFile, try again.");
            System.Environment.Exit(-1);
        }

        // WriteFile
        uint numBytes = 0;
        int response_code = 0;
        var buffer = new byte[] { 0x41, 0x0, 0x42, 0x0, 0x43, 0x0, 0x44, 0x0, 0x45, 0x0 }; // ABCDE
        IntPtr addr = Marshal.UnsafeAddrOfPinnedArrayElement(buffer, 0);
        try
        {
            response_code = WriteFile(hPipe, addr, test_string_size * 2, out numBytes, false);
        }
        catch (Exception e)
        {
            Console.WriteLine("[-] Failure calling WriteFile. Exception: {0}", e.ToString());
            System.Environment.Exit(-1);
        }
        if (response_code == 0)
        {
            Console.WriteLine("[-] Failure calling WriteFile, try again.");
            System.Environment.Exit(-1);
        }

        // Close handle
        CloseHandle(hPipe);
    }


    // Source (C++): https://pastebin.com/raw/ZsReS7k4
    public static void Main()
    {
        // CreateNamedPipe
        IntPtr hNamedPipe = CreateNamedPipe(pipe_name, PIPE_ACCESS_DUPLEX, PIPE_TYPE_MESSAGE | PIPE_READMODE_MESSAGE | PIPE_WAIT, PIPE_UNLIMITED_INSTANCES, 256, 256, 0, IntPtr.Zero);
        Console.WriteLine("[+] Handle (CreateNamedPipe): \t0x{0}", hNamedPipe.ToString("X"));
        if (hNamedPipe == IntPtr.Zero)
        {
            Console.WriteLine("[-] Failure calling CreateNamedPipe.");
            System.Environment.Exit(-1);
        }

        // CreateThread
        IntPtr hThread = CreateThread(IntPtr.Zero, 0, new ThreadStart(NamedPipe.writeFile), hNamedPipe, 0, IntPtr.Zero);
        Console.WriteLine("[+] Handle (CreateThread): \t0x{0}", hThread.ToString("X"));
        if (hThread == IntPtr.Zero)
        {
            Console.WriteLine("[-] Failure calling CreateThread.");
            System.Environment.Exit(-1);
        }

        // ConnectNamedPipe
        bool connected = ConnectNamedPipe(hNamedPipe, IntPtr.Zero);
        Console.WriteLine("[+] Named Pipe Connected: \t{0}", connected);

        // WaitForSingleObject
        WaitForSingleObject(hThread, 1000);

        // Allocate memory with correct size
        IntPtr Handle = GetProcessHeap();
        IntPtr allocated_address = RtlAllocateHeap(Handle, HEAP_ZERO_MEMORY, test_string_size * 2);
        
        // ReadFile
        uint numBytes = 0;
        int response_code = ReadFile(hNamedPipe, allocated_address, (uint)test_string_size * 2, out numBytes, IntPtr.Zero);
        if (response_code == 0) {
            Console.WriteLine("[-] Failure calling ReadFile, try again.");
            System.Environment.Exit(-1);
        }
        
        // ReadFile - Result
        byte[] data = new byte[test_string_size * 2];
        IntPtr a = IntPtr.Zero;
        ReadProcessMemory(Process.GetCurrentProcess().Handle, allocated_address, data, data.Length, out a);
        String environment_vars = Encoding.Unicode.GetString(data);
        Console.WriteLine("[+] String from named pipe: \t{0}", environment_vars);

        // NpGetUserName
        allocated_address = RtlAllocateHeap(Handle, HEAP_ZERO_MEMORY, (int)MAX_USERNAME_LENGTH);
        NpGetUserName(hNamedPipe, allocated_address, MAX_USERNAME_LENGTH);
        Console.WriteLine("[+] Pointer \"UserName\": \t0x{0}", allocated_address.ToString("X"));

        // NpGetUserName - Result
        data = new byte[(int)MAX_USERNAME_LENGTH * 2];
        IntPtr b = IntPtr.Zero;
        ReadProcessMemory(Process.GetCurrentProcess().Handle, allocated_address, data, data.Length, out b);
        String username = Encoding.Unicode.GetString(data);
        int index = username.IndexOf("\x00\x00");
        username = username.Substring(0, index);
        Console.WriteLine("[+] Result:\n{0}", username);

        // Close handle
        DisconnectNamedPipe(hNamedPipe);
        CloseHandle(hNamedPipe);
    }
}
"@
Add-Type $code4
[NamedPipe]::Main()
```

![5](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/184a3e9dbdeb2474227f5526c33e1a0daceed5c5/images/callingcsharp/Screenshot_5.png)
