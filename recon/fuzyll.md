---
value: 200
category: Recon
author: Alan Huang
---

# Fuzyll

All files are lowercase with no spaces. 
Start here: [http://fuzyll.com/files/csaw2016/start](http://fuzyll.com/files/csaw2016/start)

## Solution

A quick Google search reveals the author’s twitter, where he complains about 
not realizing the Walgreens sign is red. A search for red colorblindness turns
up deuteranopia, which doesn’t work, though the lesser form of the condition, 
deuteranomaly, does.

http://fuzyll.com/files/csaw2016/deuteranomaly is a picture of green 
strawberries. Running it through an EXIV viewer reveals that it has a 
description: “CSAW 2016 FUZYLL RECON PART 2 OF ?: No, strawberries don't look 
exactly like this, but it's reasonably close. You know what else I can't see 
well? /csaw2016/<the first defcon finals challenge i ever scored points on>.”

On the author’s linkedin page, under Honors and Awards, he lists his team coming
in third in DEFCON 19. His GitHub has a repository with challenge names. Trying
each challenge from DEFCON 19 reveals the next step of the challenge:
[http://fuzyll.com/files/csaw2016/tomato](http://fuzyll.com/files/csaw2016/tomato)

Running this text through the correct encoding (Ext Alpha Lowercase) reveals the
next clue: “CSAW 2016 FUZYLL RECON PART 3 of ?: I don't even like tomatoes! 
Anyway, outside of CTFs, I've been playing a fair amount of World of WarCraft 
over the past year (never thought I'd be saying that after Cataclysm, but here
we are). The next part is at /csaw2016/<my main WoW character's name>.”

The author’s YouTube channel has a bunch of WoW videos. His username is 
“Elmrik”.
	 
[http://fuzyll.com/files/csaw2016/elmrik](http://fuzyll.com/files/csaw2016/elmrik)
describes a function to encode a string, and gives the output. 
Decoding the string: “CSAW 2016 FUZYLL RECON PART 4 OF ?: In addition to WoW 
raiding, I've also been playing a bunch of Smash Bros. This year, I competed in 
my first major tournament! I got wrecked in every event I competed in, but I 
still had fun being in the crowd. This tournament in particular had a number of
upsets (including Ally being knocked into losers of my Smash 4 pool). On stream,
after one of these big upsets in Smash 4, you can see me in the crowd with a 
shirt displaying my main character! The next part is at 
/csaw2016/<the winning player's tag>.”

You could try trawling through streams, but that’s tedious and takes a lot of 
time. Anyways, I found a reddit thread with all the “upsets” from the 
tournament: 
[https://www.reddit.com/r/smashbros/comments/4pnkud/ceo_2016_smash_4_singles_upsets_day_1/](https://www.reddit.com/r/smashbros/comments/4pnkud/ceo_2016_smash_4_singles_upsets_day_1/). 
Trying all of the winners in sequence revealed that “Jade” was the answer.

[http://fuzyll.com/files/csaw2016/jade](http://fuzyll.com/files/csaw2016/jade) 
gives “jade.gz”, containing an image file, “jade”. This image similarly has 
metadata: “CSAW 2016 FUZYLL RECON PART 5 OF 6: I haven't spent the entire year 
playing video games, though. This past March, I spent time completely away from 
computers in Peru. This shot is from one of the more memorable stops along my 
hike to Machu Picchu. To make things easier on you, use only ASCII:
/csaw2016/<the name of these ruins>.” I don’t have a better solution than to 
brute-force landmark names around Machu Picchu. Anyways, the answer is Winay 
Wayna.

[http://fuzyll.com/files/csaw2016/winaywayna](http://fuzyll.com/files/csaw2016/winaywayna) 
has the flag, finally. 

## Flag

flag{WH4T_4_L0NG_4ND_STR4NG3_TRIP_IT_H45_B33N}
