---
value: 100
category: Forensics
author: Alan Huang
---

# Evidence.zip

As part of CSAW CTF's ongoing efforts to terminate cheaters with extreme 
prejudice, we were supposed to have evidence incriminating a handful of teams 
in this zip file. I do not know what happened, but I do not think the evidence 
actually made it in there...

## Solution

%unning the zip file through strings reveals some interesting strings at 
the end of the file:

	galf
	out/rxo802ayx4
	3ht{
	out/zhatnf5maf
	1iv_
	out/50lshu1ue6
	n4i1
	out/14o803swdq
	_3w_
	out/nppjdd2wdg
	d33n
	out/e9ydsxwetg
	rf#_
	out/0eetjrxpdm
	elee
	out/otuj3jrezt
	neff
  
These segments reversed give you the flag

## Flag

flag{th3_vi11ian_w3_n33d_#freeleffen}
