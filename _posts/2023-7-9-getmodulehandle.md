---
layout: post
title: C# implementation of GetModuleHandle  
excerpt_separator: <!--more-->
---

GetModuleHandle implementation in C# using only the NtQueryInformationProcess  API call.

<!--more-->


Link: [https://github.com/ricardojoserf/GetModuleHandle](https://github.com/ricardojoserf/GetModuleHandle)

This function takes a DLL name, walks the PEB (the Ldr structure) and returns the DLL base address. 

It works like the [GetModuleHandle](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea) function so it is useful if you want to avoid using it. This implementation uses only the NtQueryInformationProcess API call.

It is the same idea than Sektor7's Malware Intermediate course by [reenz0h](https://twitter.com/reenz0h), but in that course the code is C++ and I wanted a implementation like this in C#, I could not find it so maybe this is useful for someone else.

There is a binary to test the functionality: 

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/getModuleHandle/Screenshot_1.png)


This is the code:

```cs
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;


namespace GetModuleHandle
{
    internal class Program
    {
        [DllImport("ntdll.dll", SetLastError = true)] static extern int NtQueryInformationProcess(IntPtr processHandle, int processInformationClass, ref PROCESS_BASIC_INFORMATION pbi, uint processInformationLength, ref uint returnLength);
        private struct PROCESS_BASIC_INFORMATION { public uint ExitStatus; public IntPtr PebBaseAddress; public UIntPtr AffinityMask; public int BasePriority; public UIntPtr UniqueProcessId; public UIntPtr InheritedFromUniqueProcessId; }
        // [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)] public static extern IntPtr GetModuleHandle([MarshalAs(UnmanagedType.LPWStr)] string lpModuleName);


        private static T MarshalBytesTo<T>(byte[] bytes) { 
            GCHandle handle = GCHandle.Alloc(bytes, GCHandleType.Pinned); 
            T theStructure = (T)Marshal.PtrToStructure(handle.AddrOfPinnedObject(), typeof(T)); 
            handle.Free();
            return theStructure; 
        }


        unsafe static IntPtr auxGetModuleHandle(String dll_name) {
            IntPtr hProcess = Process.GetCurrentProcess().Handle;
            PROCESS_BASIC_INFORMATION pbi = new PROCESS_BASIC_INFORMATION();
            uint temp = 0;
            NtQueryInformationProcess(hProcess, 0x0, ref pbi, (uint)(IntPtr.Size * 6), ref temp);
            IntPtr ldr_pointer = (IntPtr)((Int64)pbi.PebBaseAddress + 0x18);
            IntPtr ldr_adress = Marshal.ReadIntPtr(ldr_pointer);
            IntPtr InInitializationOrderModuleList = ldr_adress + 0x30;

            IntPtr next_flink = Marshal.ReadIntPtr(InInitializationOrderModuleList);
            IntPtr dll_base = (IntPtr) 1;
            while (dll_base != IntPtr.Zero)
            {
                next_flink = next_flink - 0x10;                
                dll_base = Marshal.ReadIntPtr(next_flink + 0x20);
                IntPtr buffer = Marshal.ReadIntPtr(next_flink + 0x50);
                String char_aux = null;
                String base_dll_name = "";
                while (char_aux != "") {
                    char_aux = Marshal.PtrToStringAnsi(buffer);
                    buffer += 2;
                    base_dll_name += char_aux;
                }
                next_flink = Marshal.ReadIntPtr(next_flink + 0x10);
                if (dll_name.ToLower() == base_dll_name.ToLower())
                {
                        return dll_base;
                }
            }
            return IntPtr.Zero;
        }


        static void Main(string[] args)
        {
            if (args.Length < 1)
            {
                Console.WriteLine("[-] Usage: GetModuleHandle.exe DLL_NAME");
                System.Environment.Exit(0);
            }
            string dll_name = args[0];
            IntPtr base_address = auxGetModuleHandle(dll_name);

            if (base_address == IntPtr.Zero)
            {
                Console.WriteLine("[-] DLL name not found");
            }
            else
            {
                Console.WriteLine("[+] Base address of {0}: \t0x{1}", dll_name, base_address.ToString("X"));
                // Console.WriteLine("[+] Base address of {0}: \t0x{1} [GetModuleHandle]", dll_name, GetModuleHandle(dll_name).ToString("X"));
            }
        }
    }
}
```
