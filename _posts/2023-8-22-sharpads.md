---
layout: post
title: SharpADS - Playing with Alternate Data Streams (ADS) using C#
excerpt_separator: <!--more-->
---

C# program to write, read, delete or list Alternate Data Streams (ADS) within NTFS.

<!--more-->


Repository: [https://github.com/ricardojoserf/SharpADS](https://github.com/ricardojoserf/SharpADS)

---------------------------------

### Write one ADS value

Create or update and ADS value. The payload can be a string, a hexadecimal value or a url to download a file:

```
SharpADS.exe write FILE_PATH STREAM_NAME PAYLOAD
```

Example using a string:

```
SharpADS.exe write c:\Temp\test.txt ADS_name1 RandomString
```

Example using a hexadecimal value (payload starts with "0x..."):

```
SharpADS.exe write c:\Temp\test.txt ADS_name2 0x4142434445
```

Example using the content of a downloaded file (payload starts with "http..." or "https..."):

```
SharpADS.exe write c:\Temp\test.txt ADS_name3 http://127.0.0.1:8000/a.bin
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/SharpADS-screenshots/Screenshot_1.png)


---------------------------------

### Read one ADS value

```
SharpADS.exe read FILE_PATH STREAM_NAME
```

Example:

```
SharpADS.exe read c:\Temp\test.txt ADS_name1
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/SharpADS-screenshots/Screenshot_2.png)


---------------------------------

### Delete one ADS value

```
SharpADS.exe delete FILE_PATH STREAM_NAME
```

Example:

```
SharpADS.exe delete c:\Temp\test.txt ADS_name1
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/SharpADS-screenshots/Screenshot_3.png)


---------------------------------

### List all ADS values

```
SharpADS.exe list FILE_PATH
```

Example:

```
SharpADS.exe list c:\Temp\test.txt
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/SharpADS-screenshots/Screenshot_4.png)



---------------------------------

### Clear all ADS values

```
SharpADS.exe clear FILE_PATH
```

Example:

```
SharpADS.exe clear c:\Temp\test.txt
```

![img](https://raw.githubusercontent.com/ricardojoserf/ricardojoserf.github.io/master/images/SharpADS-screenshots/Screenshot_5.png)



--------------------------------------------------------

### Credits

This is based on C++ code from Sektor7's [Malware Development Advanced - Vol.1 course](https://institute.sektor7.net/rto-maldev-adv1).
