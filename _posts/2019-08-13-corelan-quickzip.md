---
layout: post
title: Corelan's QuickZip write-up
date: 2019-08-13 13:57 +0100
category: Exploit Development
permalink: /blog/quickzip-corelan
---

This is my first write-up dealing with binary exploitation, covering a BOF SEH-based vulnerability that is about a decade old.

# Introduction

I'm into Windows Exploit Development, and since you are reading this article, I presume you are as well. I mean, come on, isn't it the most challenging, self-rewarding branch of cybersecurity, where you get to burn what's left of your grey matter, attempting to turn a crash in a software, into arbitrary code execution.

So I was going through [Corelan][corelanTuts]'s exploit development tutorials (which, by the way, is one of the few excellent resources on this topic), and stumbled upon his writeup (found [here](https://www.offensive-security.com/vulndev/quickzip-stack-bof-0day-a-box-of-chocolates/)) about a 0-day vulnerability he discovered and the working exploit he made. When you have read the first paragraphs, you'll notice that he keeps challenging you to figure out the next step by yourself. So I made up my mind : why not try to exploit it myself, then come back later and finish reading. That's what I did, and I ended up doing entirely differently.

In this blogpost, i'm going to narrate how I exploited a stack based buffer overflow through abusing the SEH mechanism on a Windows XP SP3 (English) box. So, read on.

# Tools used
## Get back here to list the tools used.

# Windows Box Setup

The era is 2010, Win xp days, no ASLR yet, you can still find some non-SafeSEH OS dlls around, and DEP is set to Opt-In by default.

The OS version I use is Windows XP SP3 32bit (English), you can get it from [getintopc](https://getintopc.com/softwares/operating-systems/windows-xp-professional-sp3-32-bit-iso-dec-2016-download-8012510/) (This copy could be trojanized, we'll just deny internet access once setup is complete). I installed it on VirtualBox in my Ubuntu machine.

Go ahead and download [Immunity Debugger](https://www.immunityinc.com/products/debugger/).





corelanTuts: https://www.corelan.be/index.php/category/security/exploit-writing-tutorials/
