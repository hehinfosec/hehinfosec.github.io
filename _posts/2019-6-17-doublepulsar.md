---
layout: post
title: Plug-and-Play Usermode DoublePulsar Implant!
date:       2019-06-17
permalink: /blog/doubplepulsar
category: Misc
---

Not a while ago, while doing malware research for a company as an independent security researcher, I had to find an effective, and compact In-Memory DLL injection method, that will render the piece of code I was tasked to create, fully modular. Stealth, was another desired feature. I didn't have time, nor the knowledge to make one as effective as I wanted it to be, from the ground up (lots of issues to deal with: different Windows OS versions, OS architecture, compactness at assembly level, ...), so I had to search for an existing one. **DoublePulsar** implant, part of the [ShadowBrokers's leak](https://github.com/misterch0c/shadowbroker) was the newest of them all, made by NSA gurus, and had had success in the famous [WannaCry](https://en.wikipedia.org/wiki/WannaCry_ransomware_attack) attack.

In this write-up, *i'm not going to explain how the implant works in detail, but rather, I will go through the modifications I added to the code, as well as how to use it.* By now, you will find lots of resources talking about reversing, and dissecting the DoublePulsar at assembly level. One of the many out there, is this [countercept](https://countercept.com/blog/doublepulsar-usermode-analysis-generic-reflective-dll-loader)'s great tutorial. And for you, YARA writers, you will find there a list of memory artefacts.

### Introduction
DoublePulsar is a two-component DLL Reflective Loader, composed of a usermode and kernel part. The ring zero component is not to be studied here. It's the payload injected to the target process in user land that takes care of the actual DLL load, the kernel part sets up the environment through an Asynchronous Procedure Call (APC) call.

DoublePulsar is an In-Memory DLL Loader, meaning that the DLL in question doesn't need to reside on disk, which spares you the trouble of trying to avoid old-school signature-based antiviruses. This category of DLL loaders is not new to the market, however, what sets this particular method apart, is it's ability to load a DLL as-is. Most other existing, known techniques required the DLL to be custom-built.

### What Have Changed from the Original Code
I began by going through countercept blog post I linked above, to gain a first understanding of the exceptional NSA implant. Countercept team were kind enough to share the original doublepulsar shellcode that was extracted from one of fuzzbunch eggs. You can find it in their [github](https://github.com/countercept/doublepulsar-usermode-injector) page along with a small utility that makes use of this payload to reflectively load DLLs.

Going through understanding the whole DoublePulsar code base was a pain in the neck. I decided to start with the 64bit version, and get back later to the 32bit one. So, here is, in no particular order, what I remember I've changed in the code :

1. At the time I put my hand on the binary, there was no source code available, so I had to fully re-write DoublePulsar in assembly language, **NASM** flavor. Heavily commented, with data structures and function argument names defined. Now, anybody (well, anyone with little assembly and system programming skills) can easily modify, and adapt it to its liking.

2. One of the first remarks I made when reading the article is that there was a chunk of code not zeroed out upon return, so the function can return properly, unloading the DLL and its dependent libraries as well. That is actually required if the DLL to load is small and going to return quickly. However, it was quite the contrary with my modules, with each one of them, would be run in its own thread/process. So just after converting the code to NASM format, I added the following **IF** condition right before the calling of DLL's exported function :

![Two-mode DoublePulsar]({{ "/assets/doublepulsar/fn_return_status.png" }})

To make use of the bunch of code above, your exported function must adopt one of the following prototypes, depending on whether you intend it to return or not :

```c
/*
 * 1st mode:    void _dopu_main(void);
 * 2nd mode: int _dopu_main(unsigned char *shellc_start, size_t shellc_size, unsigned char* dll_in_memory);
 *			@param shellc_start if NULL, don't do nothing.
 *			@param dll_in_memory used as HMODULE handle to FreeLibraryAndExitThread function
 *			@return Non-Zero value to go ahead with dll cleanup code, zero to leave dll in memory.
 */
```

In 1st mode, your called function is supposed to return immediately, giving DoublePulsar shellcode the chance to clean up its mess. In 2nd mode, however, the called function is given three arguments : DoublePulsar start address in memory, its size, and the location of the DLL itself, as it was copied earlier by the implant to a chosen location in the heap. In this mode, the function assumes responsiblity to clean up the shellcode.

> The exported function in your DLL to call by DoublePulsar must have a specific prototype, a small price to pay in order to get rid of that memory artifact that could give your whole stealthy operation away ;) Well, you still have the dll loaded in memory that is not linked in loaded modules list, but then, what can you do about it ;)

3. The memory that is allocated at the beginning which the DLL is copied into from the shellcode buffer was not zeroed out or freed in the original implant. Well, in my modified version, if the DLL function ever decided to return, the DLL memory region will be freed and zeroed out.

4. I have sure applied some other changes to the code but, honestly, can't remember hehe. I also removed some chunks of code that weren't used at all.

### How to use it
If you scroll down the source code to the line *1123*, you will see the following lines of code :

![dll_params]({{ "/assets/doublepulsar/chunk_params.png" }})

The first parameter must hold the size of the DLL, the second the ordinal number of the function within the DLL to be called. The important next thing to do, is to put your dll exactly after the shellcode right where it is marked **off_dll**.

After you have modifed the implant source code the way you wish, go to a terminal and enter the following command :

```nasm
nasm -fbin [-dFN_WILL_RETURN] doublepulsar_64_usermode.asm -o out.bin
```

*FN_WILL_RETURN* is the argument that controls that **if** condition, and it depends on the mode you want doublepulsar to be assembled in.

### Extra resources available with the code
While studying DoublePulsar shellcode, I made sure to document every progress along the way. As such, I make available the following resources that will for sure make your life easier while going through the assembly code to either review/use it.

- The full IDA Pro database of the original code with every important chunk of code commented, functions and subsections given meaninful names, argument offsets and data structures defined.

![IDA Pro capture]({{ "/assets/doublepulsar/ida_screenshot.png" }})

- My modified version re-written in nasm format.
- Original DoublePulsar 64bit shellcode.
- *gen_dopu.py* which is a small python script that, when given your DPulsar binary, a DLL and the ordinal to call, packs all the inputs together into a ready-to-be-loaded dll module. One way to use it, is to package the output binary in your malware, copy it somewhere in memory (the chunk of memory doesn't need to be in an executable page), and make it the `lpStartAddress` of a new thread.

~~You can find all of these resources linked in my ![github][my_gh] page.~~

### Summary
DoublePulsar was among the most important piece of code in the arsenal of the toolset leaked by Shadow Brokers, typically used by all other exploits/frameworks. It greatly simplifies the stealth distribution of additional malware without even touching the filesystem. 

> The promissed resources are going online once I clear things out with the company, with whom I did the research. For those of you who can't wait, reach out to me and maybe we can work something out.

[my_gh]:#


