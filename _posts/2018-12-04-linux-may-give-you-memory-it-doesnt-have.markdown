---
layout: post  
title: "Linux May Give You Memory It Doesn't Have"  
date: 2018-12-04T22:11:24+01:00
---

A few weeks ago a linux computer under my team's care ( a kubernetes worker node) got totally hung because it ran out of 
memory. You couldn't even login (SSH) into it. I've been trying to find out how exactly that happened and following is what 
I dug up. I'm probably wrong about a few things here.

### What if you made a really big array?
Imagine you wrote a program that created a really really big array. So big that your computer couldn't possibly have the 
space for it (like 4 Terrabytes or something), what do you think will happen? My guess was that if I did that, I'd get a 
very angry error message, some form or other of "You're Out of Memory!".
 
### Linux may give you memory it doesn't have
Turns out if you run this unreasonable memory hogging program on Linux, that's not necessarily what would happen. Chances 
are you won't get any errors! As shocking as it was for me, this is pretty standard behaviour at least on Linux and it's called 
"Memory Overcommit".

### Most programs don't use all their memory right away
By default Linux will give us at least some percentage more memory than it has (default is 50%). This is because Linux 
thinks that we probably won't use all the memory we asked for right away, most programs don't. If it gave us 
exactly as much memory we asked for, that block of memory (figure of speech) would just sit there unused for a 
while, even if other programs needed it right away. Another way of saying this is, "Memory Overcommit increases 
utilization".

### What if you actually tried to use 4TB of memory?
So better utilization sounds good but what if our bratty program actually started using that 4TB of memory? Left unchecked 
such a program could consume all available memory. None of the other programs that need more memory would get it and 
everything would just crash. Not to worry, Linux has a backup plan! It's called the OOM Killer.

### Introducing the OOM Killer
The OOM (Out Of Memory) Killer is a program that's part of Linux. Whenever a Linux system is running out of memory, the 
OOM Killer starts to kill programs to free up memory.

Now one would guess, if our memory hogging program tries to use to the 4TB it was promised, it will be killed by the OOM 
Killer before other critical programs run out of memory, right? Well, probably ≖_≖.

### The OOM killer will keep going until the system is back to normal
The thing is that the OOM Killer can't just kill the program that's using the most memory, what if it's a really important 
program? So it decides it's victim on the bases of how much memory the program is using *and* how un-important it is. In our case, 
by default our program will definitely killed but what if our program was super important? The OOM Killer would then pick another 
poor program to free up memory and if that's not enough, it would pick another one and so on. Wow, the OOM Killer is brutal 【•】_【•】.

### My guess about our Linux machine
My guess is that, that's what happened to our Linux machine. Some other important program was using more and more memory until the system 
hung up. A program even more important than SSH!

### Questions I still have!
I still have some more questions. For example, I used the array example as a metaphor to illustrate but I don't know if 
such a program would behave _exactly_ like that. I read that the default "memory overcommit" ratio is 50% and I assumed 
it meant that Linux would give you 50% more memory than it has. But that's just my assumption ¯\_(ツ)_/¯.


