---
layout: post
title: Get process handles from process name in C# 
excerpt_separator: <!--more-->
---

Get process(es) from the process name using NtGetNextProcess and GetProcessImageFileName API calls, a stealthier alternative and written in C#.

<!--more-->


Repository: [https://github.com/ricardojoserf/GetProcessByName](https://github.com/ricardojoserf/GetProcessByName)

<br>

-----------------------

It returns a list of process handles which you can use for example to get the PIDs using [GetProcessId](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-getprocessid):

![img1](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/getprocessbyname/Screenshot_1.png)

<br>

-----------------------

The code:

```cs
using System;
using System.Text;
using System.Runtime.InteropServices;
using System.Collections.Generic;

namespace GetProcessByName
{
    internal class Program
    {
        [DllImport("ntdll.dll")] static extern bool NtGetNextProcess(IntPtr handle, int MAX_ALLOWED, int param3, int param4, out IntPtr outHandle);
        [DllImport("psapi.dll")] static extern uint GetProcessImageFileName( IntPtr hProcess, [Out] StringBuilder lpImageFileName, [In][MarshalAs(UnmanagedType.U4)] int nSize );
        [DllImport("kernel32.dll")] static extern int GetProcessId(IntPtr handle);

        public static List<IntPtr> GetProcessByName(string proc_name)
        {
            IntPtr aux_handle = IntPtr.Zero;
            int MAXIMUM_ALLOWED = 0x02000000;
            List<IntPtr> handles_list = new List<IntPtr>();

            while (!NtGetNextProcess(aux_handle, MAXIMUM_ALLOWED, 0, 0, out aux_handle))
            {
                StringBuilder fileName = new StringBuilder(100);
                GetProcessImageFileName(aux_handle, fileName, 100);
                char[] stringArray = fileName.ToString().ToCharArray();
                Array.Reverse(stringArray);
                string reversedStr = new string(stringArray);
                int index = reversedStr.IndexOf("\\");
                if (index != -1) {
                    string res = reversedStr.Substring(0, index);
                    stringArray = res.ToString().ToCharArray();
                    Array.Reverse(stringArray);
                    res = new string(stringArray);
                    if (res == proc_name)
                    {
                        handles_list.Add(aux_handle);
                    }
                }
            }
            return handles_list;
        }

        static void Main(string[] args)
        {
            string proc_name = args[0];
            List<IntPtr> handles_list = GetProcessByName(proc_name);
            Console.WriteLine("[+] Name: \t{0}", proc_name);
            foreach (var proc_handle in handles_list)
            {
                int pid = GetProcessId(proc_handle);
                Console.WriteLine("[+] PID: \t{0}", pid);
            }
        }
    }
}
```
