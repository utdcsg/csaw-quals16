---
value: 150
category: Reversing
author: SneakyNachos
---

# deedeedee

Wow! I can run code at compile time! That's a pretty cool way to keep my flags 
secret. Hopefully I didn't leave any clues behind...

# Solution

Write up by: SneakyNachos

I first run the binary to get a lay of the land and get maybe an idea of what I'm dealing with.

	$ ./deedeedee 
	Your hexencoded, encrypted flag is: 676c60677a74326d716c6074325f6c6575347172316773616c6d686e665f68735e6773385e345e3377657379316e327d
	I generated it at compile time. :)
	Can you decrypt it for me?

Things to take into consideration is the clue "compile time" and the hex 
encoded-flag they spit out. Seems like a pre-generated key-gen.

First I did a quick "file" and "nm -g -C" command and came to a quick conclusion
that there was a crazy amount of functions packed into this binary.

	$ file deedeedee
	deedeedee: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.24, BuildID[sha1]=4fac9c863749015d039a3bf0a3a6c936f2f7eadd, not stripped
	
	$ nm -g -C deedeedee
	
	000000000049434c T _aaDelX
	0000000000476fa8 T _aaEqual
	00000000004770d4 T _aaGetHash
	0000000000476bc8 T _aaGetY
	0000000000476f04 T _aaInX
	0000000000494504 T _aaKeys
	0000000000476b50 T _aApplycd1
	U abort@@GLIBC_2.2.5
	0000000000489e98 T _adEq2
	0000000000489edc T __alloca
	U backtrace@@GLIBC_2.2.5
	U backtrace_symbols_fd@@GLIBC_2.2.5
	U backtrace_symbols@@GLIBC_2.2.5
	00000000006cf870 B __bss_start
	U calloc@@GLIBC_2.2.5
	U clearerr@@GLIBC_2.2.5
	U clock_getres@@GLIBC_2.2.5
	U clock_gettime@@GLIBC_2.2.5
	U close@@GLIBC_2.2.5
	...snipped...

So instead of looking through all this silliness I decided to grep for flag and 
see if there was a function that stood out.

	$nm -g -C deedeedee | grep flag | less
	
	000000000047b370 T _D2rt8typeinfo7ti_void10TypeInfo_v5flagsMxFNaNbNdNiNeZk
	0000000000497d20 T _D2rt8typeinfo8ti_float10TypeInfo_f5flagsMxFNaNbNdNiNfZk
	0000000000475968 T _D6object10ModuleInfo5flagsMxFNaNbNdZk
	00000000004807f4 T _D6object14TypeInfo_Array5flagsMxFNaNbNdNiNfZk
	0000000000475224 T _D6object14TypeInfo_Class5flagsMxFNaNbNdNiNfZk
	0000000000475788 T _D6object14TypeInfo_Const5flagsMxFNaNbNdNiNfZk
	00000000004755bc T _D6object15TypeInfo_Struct5flagsMxFNaNbNdNiNfZk
	0000000000497e78 T _D6object15TypeInfo_Vector5flagsMxFNaNbNdNiNfZk
	0000000000474ee0 T _D6object16TypeInfo_Pointer5flagsMxFNaNbNdNiNfZk
	00000000004804b8 T _D6object16TypeInfo_Typedef5flagsMxFNaNbNdNiNfZk
	00000000004809c8 T _D6object18TypeInfo_Interface5flagsMxFNaNbNdNiNfZk
	000000000048f5dc T _D6object20TypeInfo_StaticArray5flagsMxFNaNbNdNiNfZk
	0000000000475068 T _D6object25TypeInfo_AssociativeArray5flagsMxFNaNbNdNiNfZk
	0000000000474d78 T _D6object8TypeInfo5flagsMxFNaNbNdNiNfZk
	0000000000000010 D _D9deedeedee8enc_flagAya

That bottom one "_D9deedeedee8enc_flagAya" looks nice.

So let's go ahead and pop open a disassembler like radare2/Ida/binary ninja or just plain gdb and have a look at that function.

I also decided to have a quick look at main, because of the clue "ran at compile time" when executing the binary.

First I create simple test.gdb script for gdb to execute and have a peak at "main".

## Contents of test.gdb

	set height 0
	set width 0
	set disassembly-flavor intel

Then run gdb and dump main

	gdb -q deedeedee -command=test2.gdb
	Reading symbols from deedeedee...(no debugging symbols found)...done.
	(gdb) x/6i main
	0x474b08 <main>:	push   rbp
	0x474b09 <main+1>:	mov    rbp,rsp
	0x474b0c <main+4>:	mov    edx,0x44e408
	0x474b11 <main+9>:	call   0x47738c <_d_run_main>
	0x474b16 <main+14>:	pop    rbp
	0x474b17 <main+15>:	ret    

The "main" function here looks pretty suspicious of a higher-level language. 
After a quick google "_d_run_main" shows up from the programming language "D".

I guess the questions name "deedeedee" finally makes sense.

Now to dump "_D9deedeedee8enc_flagAya". I got the address "0x44cde0" from binary
ninja.

	(gdb) x/20i 0x44cde0
	0x44cde0 <_D9deedeedee7encryptFNaNfAyaZAya>:	push   rbp
	0x44cde1 <_D9deedeedee7encryptFNaNfAyaZAya+1>:	mov    rbp,rsp
	0x44cde4 <_D9deedeedee7encryptFNaNfAyaZAya+4>:	sub    rsp,0x10
	0x44cde8 <_D9deedeedee7encryptFNaNfAyaZAya+8>:	mov    QWORD PTR [rbp-0x10],rdi
	0x44cdec <_D9deedeedee7encryptFNaNfAyaZAya+12>:	mov    QWORD PTR [rbp-0x8],rsi
	0x44cdf0 <_D9deedeedee7encryptFNaNfAyaZAya+16>:	mov    rdx,QWORD PTR [rbp-0x8]
	0x44cdf4 <_D9deedeedee7encryptFNaNfAyaZAya+20>:	mov    rax,QWORD PTR [rbp-0x10]
	0x44cdf8 <_D9deedeedee7encryptFNaNfAyaZAya+24>:	mov    rdi,rax
	0x44cdfb <_D9deedeedee7encryptFNaNfAyaZAya+27>:	mov    rsi,rdx
	0x44cdfe <_D9deedeedee7encryptFNaNfAyaZAya+30>:	call   0x451470 <_D9deedeedee21__T3encVAyaa3_313131Z3encFNaNfAyaZAya>
	0x44ce03 <_D9deedeedee7encryptFNaNfAyaZAya+35>:	mov    rdi,rax
	0x44ce06 <_D9deedeedee7encryptFNaNfAyaZAya+38>:	mov    rsi,rdx
	0x44ce09 <_D9deedeedee7encryptFNaNfAyaZAya+41>:	call   0x458280 <_D9deedeedee21__T3encVAyaa3_323232Z3encFNaNfAyaZAya>
	0x44ce0e <_D9deedeedee7encryptFNaNfAyaZAya+46>:	mov    rdi,rax
	0x44ce11 <_D9deedeedee7encryptFNaNfAyaZAya+49>:	mov    rsi,rdx
	0x44ce14 <_D9deedeedee7encryptFNaNfAyaZAya+52>:	call   0x458358 <_D9deedeedee21__T3encVAyaa3_333333Z3encFNaNfAyaZAya>
	0x44ce19 <_D9deedeedee7encryptFNaNfAyaZAya+57>:	mov    rdi,rax
	0x44ce1c <_D9deedeedee7encryptFNaNfAyaZAya+60>:	mov    rsi,rdx
	0x44ce1f <_D9deedeedee7encryptFNaNfAyaZAya+63>:	call   0x458430 <_D9deedeedee21__T3encVAyaa3_343434Z3encFNaNfAyaZAya>
	0x44ce24 <_D9deedeedee7encryptFNaNfAyaZAya+68>:	mov    rdi,rax
	...snipped...

![deedeedee1](/images/deedeedee1.png)

Well would you look at that 500 something encrypt functions. You can see the 
assembly in a nicer format in img1.png of the first encrypt function.

The premise of the first encrypt function is a such.

	1.)Take in the string and size as arguments arg1 and arg2.
	2.)Using a number, in this case "111", we convert the number to a Cycle. Which can be used in "D" like an array.
	3.)Take the string argument and convert to a cycle.
	4.)for loop over the size of the string argument cycle from step (3) and xor encode with the number cycle from step (2).
	5.)Then pop of the new xor encoded value and save into the temporary space.

So basically they have 500 xor based functions that each have their own string to
xor against the flag.

At this point I wasn't recreating 500 freaking xor functions so I decided to find
a route around having to do that.

Looking back at the output of the program there must be a function that is doing 
the hex encoding of a string. My first thought was to steal that space of memory
where the encrypted flag string was located at and the size. This would allow me
to just fix up a gdb script with all the arguments replaced with the encrypted
string and then change the code path to have the encrypted flag xor'd through all
the functions.

Let's grep for hex in the functions.

	$ nm -g -C deedeedee | grep hex
	0000000000498830 T _D4core8demangle8Demangle9ascii2hexFaZh
	000000000044e370 T _D9deedeedee9hexencodeFAyaZAya

That function "_D9deedeedee9hexencodeFAyaZAya" looks good. Looks like the 
function resides at 0x44e370, after analyzing the true main in gdb.

	(gdb) x/5i main
	0x474b08 <main>:	push   rbp
	0x474b09 <main+1>:	mov    rbp,rsp
	0x474b0c <main+4>:	mov    edx,0x44e408
	0x474b11 <main+9>:	call   0x47738c <_d_run_main>
	0x474b16 <main+14>:	pop    rbp
	(gdb) x/20i 0x44e408
	0x44e408 <_Dmain>:	push   rbp
	0x44e409 <_Dmain+1>:	mov    rbp,rsp
	0x44e40c <_Dmain+4>:	sub    rsp,0x20
	0x44e410 <_Dmain+8>:	mov    QWORD PTR [rbp-0x18],rbx
	0x44e414 <_Dmain+12>:	mov    ecx,0x49f7a6
	0x44e419 <_Dmain+17>:	mov    eax,0x24
	0x44e41e <_Dmain+22>:	mov    rdx,rax
	0x44e421 <_Dmain+25>:	mov    QWORD PTR [rbp-0x10],rdx
	0x44e425 <_Dmain+29>:	mov    QWORD PTR [rbp-0x8],rcx
	0x44e429 <_Dmain+33>:	mov    rdx,QWORD PTR fs:0x0
	0x44e432 <_Dmain+42>:	mov    rbx,QWORD PTR [rip+0x274ba7]        # 0x6c2fe0
	0x44e439 <_Dmain+49>:	mov    rdi,QWORD PTR [rdx+rbx*1]
	0x44e43d <_Dmain+53>:	mov    rdx,QWORD PTR [rdx+rbx*1+0x8]
	0x44e442 <_Dmain+58>:	mov    rsi,rdx
	0x44e445 <_Dmain+61>:	call   0x44e370 <_D9deedeedee9hexencodeFAyaZAya>
	0x44e44a <_Dmain+66>:	mov    rdi,rax
	0x44e44d <_Dmain+69>:	mov    rcx,QWORD PTR [rbp-0x8]
	0x44e451 <_Dmain+73>:	mov    rsi,rdx
	0x44e454 <_Dmain+76>:	mov    rdx,QWORD PTR [rbp-0x10]
	0x44e458 <_Dmain+80>:	call   0x472f78 <_D3std5stdio20__T7writelnTAyaTAyaZ7writelnFNfAyaAyaZv>

Dumping the function "_D9deedeedee9hexencodeFAyaZAya"

	(gdb) x/20i 0x44e370
	0x44e370 <_D9deedeedee9hexencodeFAyaZAya>:	push   rbp
	0x44e371 <_D9deedeedee9hexencodeFAyaZAya+1>:	mov    rbp,rsp
	0x44e374 <_D9deedeedee9hexencodeFAyaZAya+4>:	sub    rsp,0x50
	0x44e378 <_D9deedeedee9hexencodeFAyaZAya+8>:	mov    QWORD PTR [rbp-0x48],rbx
	0x44e37c <_D9deedeedee9hexencodeFAyaZAya+12>:	mov    QWORD PTR [rbp-0x10],rdi
	0x44e380 <_D9deedeedee9hexencodeFAyaZAya+16>:	mov    QWORD PTR [rbp-0x8],rsi
	0x44e384 <_D9deedeedee9hexencodeFAyaZAya+20>:	mov    ecx,0x6cf8a0
	0x44e389 <_D9deedeedee9hexencodeFAyaZAya+25>:	xor    eax,eax
	0x44e38b <_D9deedeedee9hexencodeFAyaZAya+27>:	mov    QWORD PTR [rbp-0x40],rax
	0x44e38f <_D9deedeedee9hexencodeFAyaZAya+31>:	mov    QWORD PTR [rbp-0x38],rcx
	0x44e393 <_D9deedeedee9hexencodeFAyaZAya+35>:	mov    rdx,QWORD PTR [rbp-0x8]
	0x44e397 <_D9deedeedee9hexencodeFAyaZAya+39>:	mov    rax,QWORD PTR [rbp-0x10]
	0x44e39b <_D9deedeedee9hexencodeFAyaZAya+43>:	mov    QWORD PTR [rbp-0x30],rax
	0x44e39f <_D9deedeedee9hexencodeFAyaZAya+47>:	mov    QWORD PTR [rbp-0x28],rdx
	0x44e3a3 <_D9deedeedee9hexencodeFAyaZAya+51>:	mov    QWORD PTR [rbp-0x20],0x0
	0x44e3ab <_D9deedeedee9hexencodeFAyaZAya+59>:	mov    rbx,QWORD PTR [rbp-0x20]
	0x44e3af <_D9deedeedee9hexencodeFAyaZAya+63>:	cmp    rbx,QWORD PTR [rbp-0x30]
	0x44e3b3 <_D9deedeedee9hexencodeFAyaZAya+67>:	jae    0x44e3f6 <_D9deedeedee9hexencodeFAyaZAya+134>
	0x44e3b5 <_D9deedeedee9hexencodeFAyaZAya+69>:	mov    rdx,QWORD PTR [rbp-0x28]
	0x44e3b9 <_D9deedeedee9hexencodeFAyaZAya+73>:	mov    rax,QWORD PTR [rbp-0x30]

One spot stood out to me "0x44e39b". This spot was when the arguments decided to
flow back into the calling convention (rdx and rax). Hijacking this spot should
allow me to use the memory region where they stored the string as the argument
for the xor functions.

Let's make sure this is the case. I now have added a breakpoint and some prints
to the script "test.gdb"

	set height 0
	set width 0
	set disassembly-flavor intel
	b* 0x44e39b
	commands 1
		x/s $rdx
		p $rax
	end

Now to see if the assumptions were correct.

	$ gdb -q deedeedee -command=test3.gdb
	Reading symbols from deedeedee...(no debugging symbols found)...done.
	Breakpoint 1 at 0x44e39b
	(gdb) r
	Starting program: /home/sneaky/Documents/csaw2016/re150/deedeedee 
	[Thread debugging using libthread_db enabled]
	Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
	
	Breakpoint 1, 0x000000000044e39b in deedeedee.hexencode(immutable(char)[]) ()
	0x49f775:	"gl`gzt2mql`t2_leu4qr1gsalmhnf_hs^gs8^4^3wesy1n2}"
	$1 = 48

Cool the string seems to be loaded into rdx and the size into rax.

Now we just pick a spot at the top of the encryption function and let it run 
from there and then catch the string at the bottom.

The spot chosen is location "0x44cdf8". Which looks like the following below.

	(gdb) x/5i 0x44cdf8
	0x44cdf8 <_D9deedeedee7encryptFNaNfAyaZAya+24>:	mov    rdi,rax
	0x44cdfb <_D9deedeedee7encryptFNaNfAyaZAya+27>:	mov    rsi,rdx
	0x44cdfe <_D9deedeedee7encryptFNaNfAyaZAya+30>:	call   0x451470 <_D9deedeedee21__T3encVAyaa3_313131Z3encFNaNfAyaZAya>
	0x44ce03 <_D9deedeedee7encryptFNaNfAyaZAya+35>:	mov    rdi,rax
	0x44ce06 <_D9deedeedee7encryptFNaNfAyaZAya+38>:	mov    rsi,rdx

This piece of the encrypt flag code would pass off the arguments correctly and
then let the string and size slide through the rest of the encrypt functions.

Now time for a new test.gdb to reflect the altered code path and the
catching/printing of the final decrypted string.

	set height 0
	set width 0
	set disassembly-flavor intel
	b* 0x44e39b
	commands 1
		printf "Hex encoder end case"
		x/s $rdx
        p $rax
        set $pc = 0x44cdf8
	end

	b* 0x44e369
	commands 2
        x/s $rdx
        p $rax
	end
	
	r

The new changes to test.gdb has the program jump to 0x44cdf8 as seen in the 
command "set $pc = 0x44cdf8", and the final print could be anywhere at the bottom
so "0x44e369" was chosen to be a good stopping point.

Now lets run and see.

	$ gdb -q deedeedee -command=test.gdb
	Reading symbols from deedeedee...(no debugging symbols found)...done.
	Breakpoint 1 at 0x44e39b
	Breakpoint 2 at 0x44e369
	[Thread debugging using libthread_db enabled]
	Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
	
	Breakpoint 1, 0x000000000044e39b in deedeedee.hexencode(immutable(char)[]) ()
	Hex encoder end case0x49f775:	"gl`gzt2mql`t2_leu4qr1gsalmhnf_hs^gs8^4^3wesy1n2}"
	$1 = 48
	(gdb) c
	Continuing.

	Breakpoint 2, 0x000000000044e369 in deedeedee.encrypt(immutable(char)[]) ()
	0x7ffff7ee7c80:	"flag{t3mplat3_met4pr0gramming_is_gr8_4_3very0n3}"
	$2 = 48

Huzzah re150 is complete.

# Flag

	flag{t3mplat3_met4pr0gramming_is_gr8_4_3very0n3}
