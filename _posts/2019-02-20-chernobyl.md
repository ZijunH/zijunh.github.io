---
layout: post
title: "Microcorruption - Chernobyl: Write-up"
date: 2019-02-21 21:52:34.000000000 +8:00
excerpt: A write-up on the solution for the microcorruption level Chernobyl. The thought process behind the solution is also included. At the end a 69 byte (nice) solution with 229009 CPU cycle time is achieved.
tags: 
  - microcorruption
  - solution
  - wargame
keywords: [microcorruption, chernobyl, solution, tip, answer, ctf, wargames, 229009, clock cycle, 69, input]
category: 
  - microcorruption
---

Before discussing the solutions, I would like to say this question was a blast. I obtained the straight forward solution relatively quickly but shortening the input size took quite a lot of time. I attempted 4-5 different possible methods, but they were all a little off at the end, preventing me from successfully opening the lock. The complexity (and length) of the code warrants many ways of attacking the problem (as evident in the varying input length), making it the best challenge (so far). Though my input size is already quite small, someone has a 60 byte solution, meaning there are still ways to attack the problem that I haven't explored.

## The input

The program accepts one string with length 0x550. The input is operated by the following rules:

- The input is separated into different commands by ';'
- Each command can start with 'a' or 'n'. Anything else will throw an error and cause early termination of the program
- Preprocessing of the input converts all spaces (0x20) to 0x0, which is used to separate command arguments
- Both types of commands take 2 arguments: "username" and "pin". Arguments after the first 2 will be ignored 
- If the command starts with 'a', arguments will be read starting from the 8th character
- If the command starts with 'n', arguments will be read starting from the 5th character
- The high bit of the pin cannot be set
- No duplicate usernames allowed

The input buffer has a size of 0x600, which is filled with 0x0. The buffer is wiped after every input, preventing a simple overflow attack using the input.

## Initial overview

As the entire program is quite complex, a good overview of the various function calls and operations is required. We can copy all the instructions into the file `initial.txt`, and then `grep` it to identify all function calls. This can be performed efficiently using the following bash command, producing the results in `final.txt`:

```
cat initial.txt | egrep '^(\w{4} )|(call)' | tee final.txt
```

I have collected the important parts of the output:

```
4438 <main>
443a:  b012 664b      call  #0x4b66 <run>

4778 <create_hash_table>
478c:  b012 7846      call  #0x4678 <malloc>
47ae:  b012 7846      call  #0x4678 <malloc>
47b8:  b012 7846      call  #0x4678 <malloc>
47e8:  b012 7846      call  #0x4678 <malloc>

4832 <add_to_table>
4866:  b012 d448      call  #0x48d4 <rehash>
4870:  b012 0e48      call  #0x480e <hash>

48d4 <rehash>
490a:  b012 7846      call  #0x4678 <malloc>
4914:  b012 7846      call  #0x4678 <malloc>
493e:  b012 7846      call  #0x4678 <malloc>
4988:  b012 3248      call  #0x4832 <add_to_table>
499e:  b012 1c47      call  #0x471c <free>
49ae:  b012 1c47      call  #0x471c <free>
49b4:  b012 1c47      call  #0x471c <free>

49cc <get_from_table>
49de:  b012 0e48      call  #0x480e <hash>
4a0a:  b012 7c4d      call  #0x4d7c <strcmp>

4b66 <run>
4b7c:  b012 7847      call  #0x4778 <create_hash_table>
4b86:  b012 504d      call  #0x4d50 <puts>
4b8e:  b012 504d      call  #0x4d50 <puts>
4b96:  b012 504d      call  #0x4d50 <puts>
4bb6:  b012 404d      call  #0x4d40 <getsn>
4c0c:  b012 cc49      call  #0x49cc <get_from_table>
4c90:  b012 cc49      call  #0x49cc <get_from_table>
4c9c:  b012 504d      call  #0x4d50 <puts>
4caa:  b012 4844      call  #0x4448 <printf>
4cb8:  b012 3248      call  #0x4832 <add_to_table>
4cc2:  b012 504d      call  #0x4d50 <puts>
```

## The storage

Each username and pin are stored in a hash table in the heap. Similar to [algiers](2019/01/algiers/), the heap is implemented as a doubly linked list. Each element of the list has the following structure:

```
<element>
    <6 byte header>
        <2 byte pointer to the previous block>
        <2 byte pointer to the next block>
        <2 byte size, with the least significant bit being indicating IN_USE>
    <content>
```

Knowing hash tables are used, it should be clear hash collisions are possible. With only 8 bins, the probability of it happening is extremely likely. In this example, hash collisions are resolved by chaining: elements with the same hash are appended at the end of a list. The memory area used by the hash table can be categorised into 2 sections: metadata and content. Initially, the hash table has the following structure, with each section belonging to a different block in the heap linked list:

```
<metadata>
    <10 byte misc values, including maximum size, addr of other metadata>
    <16 byte addr of the bins>
    <16 byte number of elements in each bin>
<content>
    <88 byte bin 0>
    <88 byte bin 1>
    <88 byte bin 2>
    <88 byte bin 3>
    <88 byte bin 4>
    <88 byte bin 5>
    <88 byte bin 6>
    <88 byte bin 7>
```

![Initial form of hash table](/assets/images/pic2.png)
*Figure showing initial form of the hash table. Red boxes represent header of the heap blocks*

## hash

Allocating the username and pin combination to the bins involve hashing the username using the hash function, which is implemented as follows:

![Implementation of the hash function](/assets/images/pic3.png)
*Figure showing implementation of the hash function*

The hash function only used the first 15 characters of the username to produce a 4 digit hexadecimal number, which is moduloed by the number of bins (8 initially). The bin allocation can be summarised using the following python code:

{% highlight python %}
def hashVal(val):
    val = list(val)[::-1]
    su = 0
    for i in range(len(val)):
        su += ord(val[i]) * (0x1f ** (i + 1))
    return "%.4x"%(su % 0x10000 % 0x8)
{% endhighlight %}

The pin is hashed in a similar way, but with additional modifications. However, I did not look into that algorithm, as it is not needed for the solution. No matter how long the inputted pin is, the hashed result will always be 2 bytes long.

After the bin number is calculated, the first 15 characters of the username is appended at the end of the chain for that bin, followed by a null byte (indicating end of string) and the hashed result of the pin. Notice both the username and pin length are limited by the algorithm, preventing any sort of overflow attack using the username and pin alone.

## rehash

The hash table becomes full when the total number of usernames and pins exceed a set size. Initially, this size is set to 11, allowing 11 usernames. Rehashing is triggered after the input of the 12th username, but before its allocation in the table. The program checks the input is valid and stores it temporarily in the input buffer. Rehashing creates a new hash table with double the number of bins using `malloc`. Then it allocates the original values to the new bins, and `free`s the old ones. The hashing result remains the same, but it will now be moduloed by a different number, thus achieving the goal of redistributing the values into different bins.

The exact implementation of `rehash` is not significant, but the function call to `malloc` before `free` is.

## malloc in rehash

`malloc` creates a specified size of free memory from the heap linked list. It starts at a designated block of the linked list and checks if the block is in use by the `IN_USE` bit specified in `size` of the block header. 

If the block is in use, the function skips to the next block using `.next` of the header. This process is repeated until one of the following two possible events happen:  `.next` points to a blocker that with a lower address compared to the current, or `.next` points to the initial block. These two events indicates `malloc` has probed all blocks within the linked list (second event is special case for 1 memory block only, where `.next` points to itself), and the program crashes with the error "heap exhausted".

If the block is not in use, its size is checked. When the size of the block is the same as the specified size, the `IN_USE` bit is set and the function returns the address of the block. If the size is not the same, the empty block will be divided into 2, with the lower-addressed block labelled as `IN_USE` with the required size. This block points to the other block through `.next`, and the header of the blocks are updated accordingly.

rehash calls `malloc` multiple times, and most of the time the search starts at 0x5000 (start of the heap).

## free in rehash

`free` removes a block of used memory and attempts to merge it with the proceeding or preceding not in use block. As seen in [algiers](2019/01/algiers/), it can be summarised by the following code:

```
r15 = &freeing chunk
r13 = r15.size
r14 = r15.prev
r12 = r14.size
if (!r14.IN_USE){
    r12 += 6
    r12 += r13 
    r14.size = r12
    r14.next = r15.next
    r13 = r15.next
    r13.prev = r14
    r15 = r15.prev
}
r14 = r15.next
r13 = r14.size 
if (!r14.IN_USE){
    r13 += r15.size
    r13 += 6
    r15.size = r13
    r14.next = r15.next
    r14.prev = r15
}
```

The locations that will be `free`d are obtained from the hash table's metadata, which are the addresses of the bins.

It is important to notice `malloc` and `free` obtain the bin addresses through different methods. While `malloc` uses the `.next` in headers, `free` uses pre-defined addresses.

## The exploit

The exploit depends on the 6th username in each bin overwriting the header of the next bin. The overwriting process starts at the 1st character, requiring no additional padding. Effectively, this allows control of the `.prev`, `.next`, and the `.size` attributes. The 11th username in the bin can also overwrite the header of the bin after the next. However, only the 7th character in the username can do so, thus requiring a 6-character padding within the username. With full control of a block header, we can use `free` to overwrite at least one byte in the program, which leads to many different possibilities of attack. Looking at the call structure of the program, it appears `free` is only called in `rehash`, which occurs after the input of the 12th username but before its allocation.

The outline of the plan of attack simple: 

1. Input 5 dummy usernames that belong to the same bin2
2. Input a specifically crafted string that overwrites the next header
3. Input 5 more dummy usernames
4. Input 1 dummy username that triggers `free` in `rehash`

   
Voila! Now we should have some control of the flow of the program, allowing many different routes of attack. However, the exact details are difficult to resolve, and I will detail them in the following sections.

### Hash collision

Control exactly which bin each username and pin combination go to is required for the exploit to work. As there are only 8 bins, collisions are extremely likely. Since the hashing function is already reverse engineered, I wrote a script that calculates the bin location for all 1 character usernames, which will be used as our dummy usernames. The hex encoding of the characters can be obtained by `readStrs(res[binNo])`.

{% highlight python %}
import itertools

STR_LENGTH = 1

def hashVal(val):
    val = list(val)[::-1]
    su = 0
    for i in range(len(val)):
        su += ord(val[i]) * (0x1f ** (i + 1))
    return su % 0x8

def readStrs(val):
    for i in val:
        for chr in i:
            print("%.2x"%ord(chr), end="")
        print()

strs = "".join([chr(i) for i in range(0x100)])
res = {}
pos = map(''.join, itertools.product(strs, repeat=STR_LENGTH))

for i in pos:
    val = hashVal(i)
    if(val not in res):
        res[val] = []
    res[val].append(i)
{% endhighlight %}

### Which bin?

It is natural to wonder which bin header our sixth input should overwrite. Careful inspection of `free` reveals that if the previous block has the `IN_USE` turned off, `.prev` of the block pointed by `.next` will be overwritten. This may overwrite the carefully crafted header we created if the block at addr `.next` is the block with the overwritten header. This is guaranteed to happen in every block after the first, as `free` occurs from bin 0 to the last bin, clearing `IN_USE` sequentially. To prevent this, we have to overwrite the header of bin 1, as `.prev` of bin 0 pointed to the hash table metadata, which will definitely be in use. To overwrite the header in bin 1, the first 6 inputs must be placed in bin 0.

### How to overwrite at the desired location?

Many restrictions apply when overwriting the header due to checks in the program. Most of the checks are performed by `malloc`, which has strict rules on the value stored in `.next`. The simple (one I initially tried) solution utilises the 11th input to overwrite a second header for the block. The first overwrite changes the `.next` of the header to point it towards the third block downstream. This hides the second block downstream from `malloc`, bypassing its checks. The `.prev` of the second block is set to a return pointer location on the stack, `.next` to a custom shell code location. With this, we can successfully unlock the door. However, this requires a high input length due to the custom shell code and extra padding size required for the second input, so I attempted to find an exploit that only requires one overwritten the header.

When overwriting the header, the location we want to overwrite is stored in either `.prev` or `.next`. Storing it in `.prev` requires the 2 byte value at 0x4(`.prev`) to be even (not `IN_USE`) to trigger the required `if` statement correctly. Storing it in `.next` means the address must be larger than the current address, or `malloc` will crash the program. As the address we need to overwrite is often the program code or a value in the stack, the latter case is highly unlikely, suggesting `.prev` must hold the address we want to overwrite. 

## What to overwrite?

As `.prev` will be used to store the address, the first `if` statement in `free` will be triggered. Thus, the value we want write will be stored at `.next`, which must follow the `malloc` rules: `.next` must be after current block address. The value must also point to a address with a valid block header with sufficient size for `malloc` to use or a `.next` that points to a valid chain. This limits the possible values to be approximately  between 0x5000 and 0x5300 (the range our input will be stored at and the range of available block headers).

Our selected value can overwrite one of two things: a return address on the stack or a part of program code. Unfortunately, the former method is not possible. None of the functions call `INT` with 0x7f, preventing us from directly opening the door. In addition, the first `if` statement in `free` also copies the `.prev` of the current block and replaces the `.prev` of the block pointed by `.next`. The stack is also placed between 0x3dd0 and 0x3dff, of which the '3d' prefix refers to a jump statement, changing the program counter (pc) by a large fixed amount. Thus, if we do use the first method, the return address on the stack will be replaced, the pc will jump to the value we refer to, which has the first 2 bytes replaced by a value between the range 0x3dd0 and 0x3dff, causing the pc to jump again, missing our custom shellcode.

Therefore, only the latter method is possible: we must overwrite a part of the program code. Through many trial-and-error attempts and careful observations of the program, I found a potential target in the program: the instruction "add #0x600, sp" at 4cc8. This instruction moves the stack pointer by 0x600, which moves it from the start of the input buffer to the end. By overwriting this instruction, we can prevent movement of the stack pointer, and with a special input, we can direct the return pointer to a arbitrary location (this time without any checks on the input as it is still in buffer).

I planned to direct the return pointer to the `INT`. With the control of the stack, we can provide the function with any argument we want. For `INT`, an argument of 0x7f opens the lock automatically. The argument is placed 2 bytes behind the return address due to the implementation in the program, so a 2 byte padding is required.

Unfortunately, the instruction at 4cc8 is only reached when an invalid command is entered (not starting with 'a' or 'n'), so a 12th input is required to trigger that section of code. Due to the `pop` instructions before the `ret`, the stack pointer will move further downstream from the start of the input buffer. This requires some extra padding in the input buffer.

![Instructions near 4cc8](/assets/images/pic3.png)
*Figure showing instructions near 4cc8*

## The solution

The solution requires 12 consecutive commands that each contain a username allocated to bin 0:

5 dummy commands
1 command with the form <--8 byte "new "--><--2 byte '.prev'--><--2 byte '.next'--><--2 byte size--><--2 byte " " + pin-->
5 dummy commands
1 dummy command that can belong to any bin
1 invalid command that has the form <--padding--><--2 byte address of `INT`--><--2 byte padding--><--7f-->

The dummy commands have the form <--8 byte "new "--><--1 byte usrname--><--2 byte " " + pin-->

The following inputs (separated by line) requires 95 bytes of input, and they successfully unlock the door:

```
6e657720082041
6e657720182041
6e657720682041
6e657720282041
6e657720302041
6e657720c64cfc50f5f32041
6e657720382041
6e657720402041
6e657720482041
6e657720502041
6e657720582041
6e657720602041
aaaaaaaaaaaaaaaaec4caaaa7f
```

### Decreasing the length

The solution above, however, is not the optimal solution. Experimentation with the pins reveals 0x0 is a valid pin. As the pins play no significant role (from what I found), their values do not matter. One may also remember the input is preprocessed by the program, replacing all 0x20 bytes with 0x0. The above two factors imply the command "6e657720082041" play an equivalent role as "6e657720082000", which is equivalent to "6e657720080000". As the input buffer is guaranteed to be filled with 0x0 (it is cleared after each input), the last 2 bytes of the command may be omitted.

Further optimisation can be performed by omitting all bytes after the initial '6e'. As the program confirms the command by the initial character instead of the entire string, omitting the remaining values tricks the program into thinking a username of 0x0 was entered. Furthermore, a username of 0x0 conveniently has a hash that allocates it to bin 0, allowing it to be used. However, as duplicate usernames are not allowed, this command can only be used once.

Concatenation of the last 2 commands can be performed as the length of the padding exceeds the length of the command. The commands are separated with ';'. As the size of the previous command and ';' is the same as the padding, no further padding is required.

This allows the following inputs, which unlocks the door:

```
6e
6e65772018
6e65772008
6e65772028
6e65772030
6e657720c64cfc50f5f3
6e65772038
6e65772040
6e65772048
6e65772050
6e65772058
6e6577206020413bec4caaaa7f
```

The '.next' in the 6th command was not randomly chosen. The instruction with value 0x50fc requires 2 arguments, allowing it to include one of the `push` instructions as its argument, thus shortening the padding required by 2 bytes.

Even though this post is already very long, a lot of details are still not covered properly, such as alternative places to overwrite. I experimented with many of these, but they failed to unlock the door due to the many restrictions imposed on the input. Looking at the hall of fame, the minimum input required is only 60 bytes! If anyone is willing to share the answer, please let me know; I have wasted too much time trying to shorten the solution.


