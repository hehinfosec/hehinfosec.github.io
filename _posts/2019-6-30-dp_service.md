---
layout: post
title: DoublePulsar Free Service
category: Misc
permalink: /blog/dp_service
---

## Why does this service matter ?
The Advanced Equation Group's toolset leaked by Shadow Brokers is being more and more used by *street-level* cybercriminals, as was reported by this [trendmicro](https://blog.trendmicro.com/trendlabs-security-intelligence/advanced-targeted-attack-tools-used-to-distribute-cryptocurrency-miners/?utm_source=trendmicroresearch&utm_medium=socal) blog post earlier this month. But, taking a close look at the rudimentary, unsophisticated way, they have used the toolset (see image below), it makes no... BLAH HERE

![files_on_disk](https://blog.trendmicro.com/trendlabs-security-intelligence/files/2019/06/Equation-Group-Tool-1.png).

I'm sure the image has angered a lot of people as it did for me. A bunch of executable files (exe, dll) are dropped on disk, waiting to be detected by any installed antivirus. This is painful to watch!!!

## What does this service solve ?
DoublePulsar is one of the awesome tools that has been leaked, read my [article](https://hehinfosec.github.io/doublepulsar/) here, it allows any DLL to be loaded in memory without it touching the filesystem. Well, you got a DLL, upload it here, and get your ready to be used version.

## How to use the output binary ?
While it is in Memory, just transfer CPU execution to it, either through starting a new thread/process, or even a simple *jmp*; then watch your exported function do its magic. cryptojacking much ;)
