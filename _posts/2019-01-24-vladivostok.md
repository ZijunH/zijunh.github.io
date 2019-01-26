---
layout: post
title: "Microcorruption - Vladivostok: Write-up"
date: 2019-01-24 11:16:29.000000000 +08:00
excerpt: A write-up on the solution for the microcorruption level Vladivostok. The thought process behind the solution is also included. At the end a 17 byte solution with 101670 CPU cycle time is achieved.
tags: [microcorruption, solution, wargames]
keywords: [microcorruption, algiers, solution, tip, answer, ctf, wargames, 101670, clock cycle, 17, input]
---

## The input

For this particular problem, the user can only input two string in the form of `username` and `password`. The maximum accepted length for `username` is 0x8 bytes, and the maximum accepted length is `password` is 0x14 bytes .

## The exploit

Again, we are able to look at the function names and notice there is a function containing `aslr`. Though it looks complicated, we are going to ignore how the function works on a microscopic level, and use our knowledge of ASLR to solve the problem. ASLR stands for address space layout randomisation. It involves placing the libraries, stack, and heap (doesn't exist in this case) at random locations within memory. It serves to protect program when the exploiter controls a return pointer. As the locations are randomised, she/he will not know where the desired code fragment is stored.

However, this technique is easily countered by knowing the location of one function in memory. This way, the location of the desired code fragment can be calulated by adding the offset to the exposed location, and the offset is easily determined by looking at the machine code.

In this example, this is fortunately the case. A `printf` function exists, which can be exploited using `%x` to reveal the content on the stack. Trying with an input of `%x%x%x%x` provides an output of `0000xxxx00000000`, where `xxxx` is random, hinting this may be a pointer to another function. Pausing the program execution and comparing the assembly code with the initial state of the program, we can identify it as the start of the `printf` function.

Controlling the return pointer is easy through a buffer overflow of the input string. A small trial-and-error reveals the return pointer of the function is stored 8 bytes after the start of the input. 

As `printf` function location can be exposed in the program, we have to consider what shell code we want to inject. As the location of the stack is randomised separately, it is difficult to determine the stack loction, thus prevents use from including custom code in the input. What we do have access to, however, is the content of the stack. Therefore, we are should be able to push 0x7f, the door opening code, onto the stack and call `INT`, which opens the door.

Examination of the code reveals `INT` is 0x180 bytes after the start of `printf`. Pushing 0x7f is done by appending 2 bytes of padding and `7f` at the end of the input string.

## The solution

`username` should print a string with the following format:

<--2 bytes of 0--><--address of `printf`-->

`password` has the following format:

<--8 bytes of padding--><--`printf` address + 0x180--><--2 bytes of padding--><--7f-->

`username`:

```
%x%x
```

`password`:

```
1122334455667788c2ad11227f
```

P.S.: Apparently there exists a 14 byte and a **9** byte solution. If anyone can explain how please tell me.


## Read more

[How “leaking pointers” to bypass DEP/ASLR works](https://security.stackexchange.com/questions/22989/how-leaking-pointers-to-bypass-dep-aslr-works)