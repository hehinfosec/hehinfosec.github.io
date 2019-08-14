---
layout: post
title: Corelan's QuickZip write-up
date: 2019-08-13 13:57 +0100
category: Exploit Development
permalink: /blog/quickzip-corelan
---

This is my first write-up dealing with binary exploitation, covering a BOF SEH-based vulnerability that is about a decade old.

## Introduction
I'm into Windows Exploit Development, and since you are reading this article, I presume you are as well. I mean, come on, isn't it the most challenging, self-rewarding branch of cybersecurity, where you get to burn what's left of your grey matter, attempting to turn a crash in a software, into arbitrary code execution.

So I was going through [Corelan][corelanTuts]'s exploit development tutorials (which, by the way, is one of the few excellent resources on this topic), and stumbled upon his writeup (found [here](https://www.offensive-security.com/vulndev/quickzip-stack-bof-0day-a-box-of-chocolates/)) about a 0-day vulnerability he discovered and the working exploit he made. When you have read the first paragraphs, you'll notice that he keeps challenging you to figure out the next step by yourself. So I made up my mind : why not try to exploit it myself, then come back later and finish reading. That's what I did, and I ended up doing entirely differently.

In this blogpost, i'm going to narrate how I exploited a stack based buffer overflow through abusing the SEH mechanism on a Windows XP SP3 (English) box. So, read on.

## Tools used
**back here to list the tools used.**

## Windows Box Setup
The era is 2010, Win xp days, no ASLR yet, you can still find some non-SafeSEH OS dlls around, and DEP is set to Opt-In by default.

* The OS version I use is Windows XP SP3 32bit (English), you can get it from [getintopc](https://getintopc.com/softwares/operating-systems/windows-xp-professional-sp3-32-bit-iso-dec-2016-download-8012510/) (This copy could be trojanized, we'll just deny internet access once setup is complete). I installed it on VirtualBox in my Ubuntu machine.

* Go ahead and download [Immunity Debugger](https://www.immunityinc.com/products/debugger/). Also, get [mona](https://github.com/corelan/mona) plugin, download the .py file, and put it under **PyCommands** folder under ImmunityDebugger installation path.

* A copy of the vulnerable software, [QuickZip 4.60.019](https://www.exploit-db.com/apps/06c1cb6633edd4688cc00864a7079935-quickzip_4-60-019.exe), and install it.

We're all set.

## The vulnerability
This particular version of QuickZip is vulnerable to a stack-based buffer overrun when is fed a specially crafted zip file. The below python code builds `corelanboom.zip` file that is susceptible to crash the program.

```python
import struct

ldf_header = "\x50\x4B\x03\x04\x14\x00\x00" + \
             "\x00\x00\x00\xB7\xAC\xCE\x34\x00\x00\x00" + \
             "\x00\x00\x00\x00\x00\x00\x00\x00" + \
             "\xe4\x0f" + \
             "\x00\x00\x00"

cdf_header = "\x50\x4B\x01\x02\x14\x00\x14" + \
             "\x00\x00\x00\x00\x00\xB7\xAC\xCE\x34\x00\x00\x00" + \
             "\x00\x00\x00\x00\x00\x00\x00\x00\x00" + \
             "\xe4\x0f" + \
             "\x00\x00\x00\x00\x00\x00\x01\x00" + \
             "\x24\x00\x00\x00\x00\x00\x00\x00"

eofcdf_header = "\x50\x4B\x05\x06\x00\x00\x00\x00\x01\x00\x01\x00" + \
                    "\x12\x10\x00\x00" + \
                    "\x02\x10\x00\x00" + \
                    "\x00\x00"

payload = "A"*4064
payload += ".txt"

print "Size : %d" % len(payload)

with open('./corelanboom.zip', 'wb') as f:
    f.write(ldf_header + payload + cdf_header + payload + eofcdf_header)
```

Copy the .zip file to the WinXP box and open it with QuickZip. See picture below.

![zipboom_open]({{ "/img/zip_open.png" }})

The content of the `payload` variable becomes the name of a compressed file, when double clicked to open, crashes the application.

## From Crash to Exploit
Let's fire up ImmunityDebugger (ImmDbg from now on) and carefully study the crash. Once inside, Press **F3** and choose to run QuickZip, then **F9** to continue execution. Choose to open our malformed .zip file, and double click again the `"AAAA..."` file inside. The First chance exception kicks in ImmDbg, giving you a chance to analyze it before it is handed over to the application's stack-frame exception handler.

![AV]({{ "/img/AV.png" }})

with the faulty instruction pointing at (**EDI = 0x130000**)

![faulty_instr]({{ "/img/faulty_instr.png" }})

Press **Alt + S** to show the SEH chain of the current thread; one look at it and we see that the second entry has been corrupted with our A's.

![seh_corrupt]({{ "/img/seh_corrupt.png" }})

![seh_corrupt]({{ "/img/seh_corrupt2.png" }})

So we have to deal wit an SEH-based exploit. Now go to the stack view (the pane on the bottom-left), and select the address at *SE handler*, then scroll down to the bottom of the stack, where there are no more `A's`. Right-click on the last one, then from the drop-down menu, go to **Adress->relative to selection**. This says, that we have a space of `0x3F8 = 1016` bytes to house our shellcode after the SEH record.

Sweet!!! Couldn't get any easier than that. If SEH based exploits are new to you, read this [blogpost](https://www.corelan.be/index.php/2009/07/25/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-3-seh/) then come back :). Let's quickly craft a working exploit.

First, the offset to the SEH record. In ImmDbg's Python shell at the bottom, type `!mona pattern_create 3000`, then open `C:\Program Files\Immunity Inc\Immunity Debugger\pattern.txt` to retrieve the generated Metasploit Cyclic Pattern (variable *msf* in script)

![msf_3000]( {{ "/img/msp.png" }})

Replace **payload** content with : `payload = msf + "A"*(4064 - len(msf))`
Go ahead and generate the .zip file, load it in QuickZip while attached to ImmDbg (or you can press **Ctrl + F2** to restart the debugged program), after that, you trigger the 1st chance exception. Once there, **Alt + S** for SEH chain window to appear, and then, on the corrupt SEH entry, right-click to *Follow address in stack*.

> Sometimes you hit a user-defined exception (thrown with ntdll!RtlRaiseException) like the one shown below. In this case, all you have to do is a **Shift + F9** to pass it to the program for handling, then you'll hit the above AccessViolation exception.
> ![user_excpet]({{ "/img/user_except.png" }})

![seh_msf]({{ "/img/seh_msf.png" }})

![seh_msf2]({{ "/img/seh_msf2.png" }})

Use the 4 alphanumeric bytes next to *Pointer to next SEH record* in conjuction with *!mona* PyCommand to get the desired offset : `!mona pattern_offset Aj9A` (the result is on the *Log* view, **Alt+l**). I need to prepend the payload with **297 bytes** before *next SEH* is reached.

Second, address of a **pop/pop/ret** sequence in a non-safeseh module. `!mona seh` will take care of that. the output file *seh.txt* can be found in ImmDbg installation directory. Only **QuickZip.exe** is non-safeseh built from the application's modules, grab an address from here if you need a multi OS version exploit, although you'll have to deal with a null-byte. I choose `0x6d7e467b` from **D#DXOF.dll**, which is a system dll.





[corelanTuts]: https://www.corelan.be/index.php/category/security/exploit-writing-tutorials/
