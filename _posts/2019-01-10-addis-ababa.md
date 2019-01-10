---
layout: post
title: "Microcorruption - Addis Ababa: Write-up"
date: 2019-01-10 17:15:03.000000000 +08:00
excerpt: A write-up on the solution for the microcorruption level Addis Ababa. The thought process behind the solution is also included.
tags: [microcorruption, solution, wargames]
keywords: [microcorruption, addis ababa, solution, tip, answer, ctf, wargames]
---

## The input

For this particular problem, the user can only input one string in the form of `username:password`.

## The exploit

In the manual, it is hinted that "Usernames are printed back to the user for verification", suggested the attack origin is from the output function. The function name `printf` should also ring some bells. Both of these points the attack method towards [uncontrolled format string attacks](https://en.wikipedia.org/wiki/Uncontrolled_format_string).

However, without meta knowledge on what type of exploit we may use, this is not so obvious due to the many jumps and conditions in the `printf` function. We are still able to eliminate the possibility of an overflow attack by observing the arguments passed in the `getsn` function. Only 0x13 characters will be registered from the input, which is too short for overflows.

With a rudimentary idea of the attack, we can explore the different targets. As format string attacks often rely on `%n` to write to a memory address, we need to decide where and what to write. As there is no `ret` at the end of the `main` function, we cannot redirect the return address to the `unlock_door` function nor perform arbitrary code execution. 

At the bottom of the `main` function, it is observed that the program tests whether the value stored at `0x0(sp)` is 0 or not. If it is, the code is directed to the "invalid" part and an error message printed; if it is not, the door is unlocked. We need to change the value to a non-zero one to bypass it.

Uncontrolled format string attacks rely on the function not knowing the number of arguments. It can only guess by the number of "%" arguments. As arguments are nothing but numbers on the stack, the pointer for the arguments may overlap with the input string, if they are stored in proximity. In this challenge, this is the case.

## The solution

The start of the argument pointer is 2 bytes away from the input string, so a read will be performed to move the pointer to the start of the string by %x. Then we want to change the value stored at `0x0(sp)`. This is a fixed location and can be found in a debug session. The change is performed using "%n", which writes the number of bytes printed by printf to the memory address provided by the argument. As the argument is now the start of the string, we can just provide the required memory address at the start.

The final solution should have the following format:

<--two bytes of address-->%x%n

In my case, it is <>%x%n.

## Read More

[Exploiting Format String Vulnerabilities](https://crypto.stanford.edu/cs155/papers/formatstring-1.2.pdf)