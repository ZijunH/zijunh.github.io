---
layout: post
title: "Microcorruption - Hollywood: Write-up"
date: 2019-02-25 18:22:19.000000000 +8:00
excerpt: A write-up on the solution for the microcorruption level Hollywood. The thought process behind the solution is also included. At the end a 5 byte solution with 3085563 CPU cycle time is achieved.
tags: 
  - microcorruption
  - solution
  - wargames
keywords: [microcorruption, hollywood, solution, tip, answer, ctf, wargames, 3085563, clock cycle, 5, input]
category: 
  - microcorruption
---

## Initial information

In the manual for the lock, it claims "this lock is not attached to any hardware security module", suggesting no interrupts will be used to open the door. This implies the password must be stored in memory; if we can reverse engineer the program in debug, we can obtain the password. However, this could be difficult due to the "new hardware random number generator", which makes it "impossible to know where the password will be". This is also quite obvious looking at the extremely short and garbage-looking code that is displayed by the disassembly.

## The input

We can only enter one string for the password. I did not check for the length as the initial information suggests the only way to solve the problem is to reverse engineer the password itself.

## The exploit

Clearly, the input string will be compared with the password at some point to verify it. Therefore, we can focus on the program after the input of the string, which saves us some time and effort. As the online disassembler is not useful during the process, I recommend using a better disassembler with a scripting functionality that supports the MSP430 architecture. The only one that appears to support MSP430 is IDA pro, which is very expensive. I did not have the luxury to use it, so I will be using the online disassembler with some javascript (only disadvantage is slow) to solve this problem.

After entering the password, one can search for the string within the memory. The process must be repeated many time to ensure its storage location is not randomised as well. Eventually, we can find the input string stored at 0x2600.

Now, we must step through the program to find out what is happening. The useful instructions are selected and shown below:

```
...

3240 00a0      mov	#0xa000, sr 
b012 1000      call	#0x10

...

0c4f           mov	r15, r12
3cf0 fe0f      and	#0xffe, r12
3c50 00e0      add	#0xe000, r12

...

8c4f 0000      mov	r15, 0x0(r12)
...
8c4f 0200      mov	r15, 0x2(r12)
...
8c4f 0400      mov	r15, 0x4(r12)
...
8c4f 0600      mov	r15, 0x6(r12)
...
8c4f 0800      mov	r15, 0x6(r12)
...
8c4f 0a00      mov	r15, 0xa(r12)
...
8c4f 0c00      mov	r15, 0xc(r12)

...

004c           br	r12
```

From experimentation, calling the interrupt with 0xa000 stores a random number in r15. The program then calculated address within the range 0xe000 and 0xefff and stores it in r12. 16 bytes of information are written, starting from r12. Eventually, the program jumps to r12, where the written information is executed. From this segment of code, we can hypothesis the values stored at r12 are the unobfuscated instructions that does the password comparison. This is proved when the first instruction is `mov #0x2600, r5`; recall 0x2600 is the address of the input. The long sequences  of code we found are merely used to unobfuscated the instructions. 

To proceed with our analysis we need to obtain all the instructions that are ran at r12. This requires use to step through the massive loops of instructions that unobfuscate the code. This will take too much effort manuually, so a script is required. Even though the online debugger does not have a scripting function, we can write a small javascript program that utilises the `parse` function, which parses its first argument as a command entered in the debugger. We can then store all the instructions executed at any address that starts with 'e'. In this program, I have calculated the offsets for some important code fragments in the program to slightly speed up the program. As the debugging process is performed on the server, an 1 second delay is added to allow the information to transfer. This results in the following program:

{% highlight javascript %}
dic = []

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function runCode() {
    while(true){
        if(document.getElementById("reg0").textContent == "0010"){
            parse("s");
            await sleep(1000);

            target = parseInt("0x" + document.getElementById("reg0").textContent) + 0x4a;
            parse("break " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            parse("c");
            await sleep(1000);

            parse("s");
            await sleep(1000);

            parse("unbreak " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            target = parseInt("0x" + document.getElementById("reg0").textContent) + 0xa;

            parse("break " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            parse("c");
            await sleep(1000);

            parse("unbreak " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            target = parseInt("0x" + document.getElementById("reg12").textContent);
            parse("break " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            parse("c");
            await sleep(1000);

            parse("unbreak " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            while(document.getElementById("reg0").textContent[0] == 'e'){
                dic.push(document.getElementById("insnbytes").textContent);
                parse("s");
                await sleep(1000);
            }

            target = parseInt("0x" + document.getElementById("reg0").textContent) + 0x38;
            parse("break " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

            parse("c");
            await sleep(1000);

            parse("unbreak " + target.toString(16).padStart(4, '0'));
            await sleep(1000);

        }else{
            parse("s");
            await sleep(1000);
        }
    }
}

runCode()

for(i = 0; i < dic.length; i++){
  console.log(dic[i]);
}
{% endhighlight %}

This program runs for around 10 minutes for a input with 2 bytes, and around 30 for one with 6 bytes. The results can then be copied into a text file, where the `mov` and `br` instructions can be removed using `egrep`:

```
egrep -v "^(3c40|004d)" uncleaned.txt | tee cleaned.txt
```

Disassembling the remaining instructions provides the following code:

```
3540 0026      mov	#0x2600, r5
0643           clr	r6

//loop start
2455           add	@r5, r4
8410           swpb	r4
36e5           xor	@r5+, r6
06e4           xor	r4, r6
04e6           xor	r6, r4
06e4           xor	r4, r6
8593 0000      tst	0x0(r5)
0742           mov	sr, r7
27f3           and	#0x2, r7
0711           rra	r7
17e3           xor	#0x1, r7
8710           swpb	r7
0711           rra	r7
8711           sxt	r7
8710           swpb	r7
8711           sxt	r7
3840 184b      mov	#0x4b18, r8
08f7           and	r7, r8
37e3           xor	#-0x1, r7
37f0 aa47      and	#0x47aa, r7
0857           add	r7, r8
0743           clr	r7
0c48           mov	r8, r12
004d           br	12
//loop end

3490 b1fe      cmp	#0xfeb1, r4
0742           mov	sr, r7
0443           clr	r4
3690 9892      cmp	#0x9298, r6
07f2           and	sr, r7
0643           clr	r6
0711           rra	r7
17e3           xor	#0x1, r7
8710           swpb	r7
0711           rra	r7
0711           rra	r7
0711           rra	r7
0711           rra	r7
02d7           bis	r7, sr
```

It appears 2 values are computed from the input and are stored in r4 and r6. The registers are then compared with 2 magic values, which act as validation for the input. If it doesn't pass, a value with the `CPU_OFF` flag set to 1 is inserted into sr, which causes termination of the program. The ending condition of the loop is checked by `tst 0x0(r5)`, and everything in the loop after that command appears to be a convoluted way to continue the loop. The loop body can thus be cleaned, and transformed into:

```
2455           add	@r5, r4
8410           swpb	r4
36e5           xor	@r5+, r6
06e4           xor	r4, r6
04e6           xor	r6, r4
06e4           xor	r4, r6
```

Now we will look at the end of the program closely. The `CPU_OFF` flag is on the 5th bit from the right, and it cannot be set at the end of the program or the CPU will be turned off, requiring `reset` to reboot it. The flag is not set when r4 is exactly 0xfeb1 and r6 is exactly 0x9298. This can be derived by tracing the change of the location of the bit in r7 as it is maniputed by the various instructions such as `rra` and `swpb`.

## The solution

With knowledge of exactly how the input is validated, we can write a program that brute forces all possible values or a set of equations that solves for the input. The latter is difficult and takes effort. We have no time constraints, so brute force is what we will use. Initially, I was lazy and tried to utilise `itertools.product` in python so I dont need to generate the number myself. However, I underestimated the time required for the brute force, as the python solution took too long. Eventually, I wrote the following the C++ program, which calculates all 6 byte solution (it implicitly includes all 5, 4, 3, 2, 1 byte solutions, where the end is padded by 0x0).

{% highlight cpp %}
#include <bits/stdc++.h>

using namespace std;

int R4_MAGIC = 0xfeb1;
int R6_MAGIC = 0x9298;

bool genNext(vector<unsigned int> *cur)
{
    bool carry = true;
    for (int i = cur->size() - 1; i >= 0 && carry; i--)
    {
        if ((*cur)[i] != 0xff)
        {
            (*cur)[i] += 1;
            carry = false;
        }
        else
        {
            (*cur)[i] = 0x0;
            carry = true;
        }
    }
    return !carry;
}

int main()
{
    vector<unsigned int> cur{0, 0, 0, 0, 0, 0};
    while (genNext(&cur))
    {
        unsigned int r4 = 0;
        unsigned int r6 = 0;
        for (int i = 0; i < cur.size(); i += 2)
        {
            int r5Val = cur[i] + cur[i + 1] * 0x100;
            r4 = (r4 + r5Val) & 0xffff;
            r4 = ((r4 >> 8) + (r4 << 8)) & 0xffff;
            r6 ^= r5Val;
            r6 ^= r4;
            r4 ^= r6;
            r6 ^= r4;
        }

        if (r4 == R4_MAGIC and r6 == R6_MAGIC)
        {
            for (auto i : cur)
            {
                cout << hex << (0xff & (int)i) << " ";
            }
            cout << endl;
        }
    }

    return 0;
}
{% endhighlight %}

Running the program for around 10 minutes generates the first 6 byte solution: 

```
0 4 a5 3c f1 5b
```

Going to the "Hall of fame", we can see the shortest solution is 5 bytes only. We can modify the loop in `genNext` so the last byte is ignored. After around 30 minutes, the first solution is generated:

```
49 49 b5 de 96 0
```

The last byte is ignored and the first 5 bytes are inputted, unlocking the door.

## Some thoughts

This question is not as difficult as the previous one. The solution is a lot more blant compared to the previous question, as there is only 1 way of attacking it (unless I missed something). This question is more of a test of reverse engineering skills rather than exploitation skills, which disallows creative solutions. Nonetheless, this is still an entertaining challenge.

This question also marks the end of the microcorruption challenge series. During the process, I have learnt quite a lot about assembly code and exploitation in general. I hope the skills can assist me in day-to-day programming in the future.

## Read more

[MSP430 instruction guide](http://www.ti.com/lit/ug/slau144j/slau144j.pdf)