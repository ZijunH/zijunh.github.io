---
layout: post
title: "Microcorruption - Novosibirsk: Write-up"
date: 2019-01-11 11:33:12.000000000 +08:00
excerpt: A write-up on the solution for the microcorruption level Novosibirsk. The thought process behind the solution is also included. Many different solutions are explored including ways to acheive 8771 clock cycle and 5 byte input.
tags: 
  - microcorruption
  - solution
  - wargames
keywords: [microcorruption, addis ababa, solution, tip, answer, ctf, wargames, 8771, clock cycle, 5, input]
category: 
  - microcorruption
---

## The input

For this particular problem, the user can only input one string in the form of `username`.

## The exploit

This problem uses the same technique as the previous problem, Addis Ababa. If you haven't read the [write-up](/2019/01/addis-ababa/) for that one, I strongly suggest you do so.

There are many ways to tackle this problem. Observe the allowed input length for this problem is 0x1f4, hinting a long input string can be used for the exploit. As we know the memory will be changed after the `printf` function, we will focus on the code after the `printf` for the input.

As you may remember in previous questions, providing a the `INT` function an argument of "0x7f" will open the door automatically, and fortunately, both `putchar` and `conditional_unlock_door` have calls to `INT`. This value is also less the the allowed input length of 0x1f4, making this attack viable.

Arguments are usually passed to functions by pushing them onto the stack. The "push" instruction for `putchar` has no arguments and pushes the default value of 0x0, preventing plans to change the argument to 0x7f. `conditional_unlock_door`, however, includes a "push" with the argument 0x7e, and we are able to override this value to 0x7f by making the length of the printed string 0x7f.

## The (first) solution

This solution has 3 parts: the address to the argument of "push" in `conditional_unlock_door`, padding to make the printed portion of the string 0x7f long, %n to overwrite.

For me the answer (hex) was:

```
c8441111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111256e
```

## The (second) solution

Naturally, I went to check the hall of fame to see how others were doing. To my suprise, I found the minimal input size was only **5 bytes**!! This prompted me to explore other possible solutions. When format string vulnerabilities were discovered, a common way to increase the string length was to use %*n*s, where *n* is a number. This pads the string, and increases the output size. However, it did not work when I tried (perhaps this is not a complete replication of the C `printf`)

But when I was trying this method, I realised there was a large area of junk memory at the end of the dump, ranging from ff80 to fff0. In `printf`, %s reads a string starting from the position indicated by the argument until it reaches a null character \0, which is indicated by 0x0. This character is observed at fffe, and there are 0x7f bytes of non-null characters before it. Therefore we can direct `printf` to print these characters using %s, which prints out the junk memory, and uses it as padding.

The solution has the following components: address of starting character in the junk memory that pads the total length of the printed string to 0x7f, the address to the argument of "push" in `conditional_unlock_door`, %s to read string, %n to overwrite.

For me the answer (hex) was:

```
83ffc8442573256e
```

This is only 8 bytes!

## The (third) solution

Though we are close, we are still 3 bytes and a lot of cpu cycles away from the best solution. To be fair, I was stuck here for a while. There is a brief discussion on reddit, which you can read [here](https://www.reddit.com/r/securityCTF/comments/8726mt/microcorruption_novosibirsk_min_input_of_5/). As there are fewer cpu cycles, changing the memory must have affected the program earlier. The only possible way to solve the challenge is to provide `INT` 0x7f. The only function that calls `INT` before `conditional_unlock_door` is `putchar`. By putting a breakpoint before the call to `INT`, we can see the stack pointer is only 2 bytes away from a character in user input.

![Stack pointer and memory location](/assets/images/pic1.png)
*Figure showing memory address location, @ is user input*

As %n can only write the length of printed characters, we are limited to writing 2 or 3 characters. Using the disassembler on the website, 0200 provides the instruction "rrc sr", and 0300 provides "rrc #0x0". Both are viable instructions that do not have big influences on the execution of `INT` (sr is overwritten by r15 in `INT`). Thus, we can overwrite one of the push instructions in `putchar`, and when `INT` is called, the stack pointer points to the character we entered after overwriting, and passes it as an argument to `INT`.

This forms the basis of our solution: address pointing towards "push" in `putchar` before `INT`, %n to overwrite the insturction with "rrc", and 7f, which will be passed as argument to `INT`.

The solution is then the following (hex):

```
5245256e7f
```

As we no longer need to wait for `printf` to finish, it uses less cycles and only requires 5 bytes of input.

## Read More

[Exploiting Format String Vulnerabilities](https://crypto.stanford.edu/cs155/papers/formatstring-1.2.pdf)

