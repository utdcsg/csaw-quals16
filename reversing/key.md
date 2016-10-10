---
value: 125
category: Reversing
author: SneakyNachos
---

# Key

So I like to make my life difficult, and instead of a password manager, I make 
challenges that keep my secrets hidden. I forgot how to solve this one and it is
the key to my house... Can you help me out? It's getting a little cold out here.

NOTE: Flag is not in normal flag format.

# Solution

Write up by: SneakyNachos

The only file the question gives is "key.exe"

I decided first to drop the file into the IDA demo and have a look around.

The first thing I noticed was 

	.rdata:004052B8 00000029 C C:\\Users\\CSAW2016\\haha\\flag_dir\\flag.txt

So I went ahead and created that file in my windows vm users directory.

Might take a few annoying permission changes to allow reading the file though.

After looking at the functions in IDA for a bit it seems as though they are 
trying to hide what they are doing via virtual functions or something like that.

My initial guess is that there's really no encryption or key mangling going on.

They probably just put the string somewhere right before the check if the key is
correct or not.

Virtual functions are bit iffy though. The string is probably hidden via a
pointer in memory versus just a normal string comparison.

I decided the best place to breakpoint was in loc_401310 shown in img1.png. 
This code block is found in function sub_401100.

The function sub_11420C0 looks interesting.

![key1](/images/key1.png)
![key2](/images/key2.png)

In img2.png I stopped at this particular spot because I had interesting strings
show up in my registers. EAX had my fake flag value, but ECX had a string that I
did not input.

![key3](/images/key3.png)

In img3.png I hex dumped that memory region and the string "idg_cni~bjbfi|gsxb" 
showed up.

So then I placed that in the flag.txt

and then I recieved the congrats shown in img4.png.

![key4](/images/key4.png)

re125 complete.

# Flag

	idg_cni~bjbfi|gsxb
