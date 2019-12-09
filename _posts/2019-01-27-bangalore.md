---
layout: post
title: "Microcorruption - Bangalore: Write-up"
date: 2019-01-27 11:47:11.000000000 +08:00
excerpt: A write-up on the solution for the microcorruption level Bangalore. The thought process behind the solution is also included. At the end a 24 byte solution with 13676 CPU cycle time is achieved.
tags: 
  - microcorruption
  - solution
  - wargames
keywords: [microcorruption, bangalore, solution, tip, answer, ctf, wargames, 13676, clock cycle, 24, input]
category: 
  - microcorruption
---

## The input

For this particular problem, the user can enter one string, `password`, with a maximum length of 0x30 bytes.

## The exploit

Looking at the function names again, we see something like `make_page_executable` and `make_page_writable`. This immediately reminds us of DEP, data execution prevention. Typically, it is a system-level feature that prevents execution of certain areas of code. We consult the [lock manual](https://microcorruption.com/manual.pdf) for more information on this feature. 

> INT 0x10.  
> Turn on DEP: pages are either executable or writable but never both.  
> Takes no arguments.  
>
> INT 0x11.  
> Mark as a page as either only executable or only writable.  
> Takes two one arguments. The first argument is the page number, the second argument is 1 if writable, 0 if executable.

As interrupts codes are moved to `sr` and transformed by the commands `swpb` and `bis 0x8000`, 0x10 corresponds to 0x9000 in `sr, and 0x11 to 0x9100. The open door interrupt 0x7f corresponds to (if you haven't noticed already) 0xff00.

Each page of memory contains 0x100 bytes, and the page number is first two digits of the memory address. Looking at `set_up_protection`, we can see 0x0000 - 0x43ff are set to be write only, and 0x4400 - 0xffff are set to be execution only.

The exploit begins with a simple stack overflow. The return address of `login` is stored 0x10 bytes downstream of the beginning of the input. As 0x30 bytes are accepted, it can be easily overwritten. However, we cannot direct it to our own shellcode in the stack, as it is not executable due to DEP. 

But, the implemention of DEP is a mere annoyance, as it can be bypassed easily by calling INT 0x11 again with the appropriate arguments. This is done by referencing the appropriate parts of `mark_page_executable`. We will provide the arguments ourselves, so the `pc` will be directed to `44ba`, after the `push` instruction. Observe the input string on the stack starts on 0x3fee, so the page number will be 0x3f.

In order to call `mark_page_executable` properly, we need to provide 2 arguments: the page number (0x3f) and the "set writable" flag (0x0). Remember the arguments are pushed in reverse order, so 0x0 will be pushed first. After the arguments, we can set the return address of this call. It will be set to 0x3fee, the start of our payload.

As mentioned before, the `sr` value for open door is 0xff00. This value will be moved to the `sr` register and `__trap_interrupt` can be called. This is done with the following assembly code:

```
mov 	#0xff00, sr
call	#0x10
```

This turns into 324000ffb0121000 for machine code. This should open the door successfully.

## The solution

The solution will have the following structure:

<--8 byte machine code payload-->  
<--8 byte padding-->  
<--2 byte DEP unlock address-->  
<--4 byte arguments-->  
<--2 byte payload address-->

and the following solution:

```
324000ffb0121000aaaabbbbaaaabbbbba443f000000f03f
```