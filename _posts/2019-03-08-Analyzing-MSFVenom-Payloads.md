The next assignment for the SLAE certification was to analyze different Metasploit `msfvenom` payloads. The three that I chose to look into were:

- linux/x86/exec
- linux/x86/chmod
- linux/x86/read_file


These payloads seemed the most interesting to me, and ones that I would default to during an engagement if I had to chose any of the `x86` payloads that weren't shells. 



Exec Payload
-------------

For the generation of these payloads, I will be utilizing the `msfvenom` command, included with the Metasploit suite of tools. `msfvenom` allows you to create payloads of all different types, OS, encodings, and much more, in a variety of different formats.

The three payloads being analyzed in this blog are fairly basic ones, but very important, and will be good examples to see how `msfvenom` works.

The first thing I will look at is the options for the `linux/x86/exec` using the command `msfvenom -p linux/x86/exec --list-options` as seen below:

```sh
Options for payload/linux/x86/exec:
=========================


       Name: Linux Execute Command
     Module: payload/linux/x86/exec
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    vlad902 <vlad902@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
CMD                    yes       The command string to execute

Description:
  Execute an arbitrary command
```

As mentioned before, this is a fairly basic payload that only requires one option `CMD`.

The following command will generate the payload for us, that will execute the `id` command, and we redirect it inside of a file:

`sudo msfvenom -p linux/x86/exec CMD=id -f c > exec_shellcode.c`

Looking at the contents of this file, we will see something familiar:

```c
unsigned char buf[] = 
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x03\x00\x00\x00\x69"
"\x64\x00\x57\x53\x89\xe1\xcd\x80";
```

So we see the shellcode that we are used to seeing, but take note that it put it into a char variable called `buf`. It did this because I provided the switch `-f c` which tells `msfvenom` to take the payload it creates and provide it within the format of a C program. 

Now that we have this, let's create a C program that will take this payload and execute it on the target system:

```c
#include <stdio.h>
#include <string.h>


unsigned char buf[] =  
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x03\x00\x00\x00\x69\x64\x00\x57\x53\x89\xe1\xcd\x80";

main () {


        printf("Shellcode Length:  %d\n", strlen(buf));
        int (*ret)() = (int(*)())buf;
        ret();

}
```
This program will also print off the size of the payload, to let us know how much of an impact we create on the target as well.

Now that we have the code to execute, we will compile as we have in previous blog posts using `gcc -fno-stack-protector -z execstack exec_shellcode.c -o exec_shellcode`. 

Now that we have an executable, let's take a look through a debugger to see what it's actually doing.


Debugging Shellcode Executable
------------------------------

So we will run the `exec_shellcode` binary with `gdb`, as this was the primary debugging tool used within the SLAE course.

```sh
$ gdb ./exec_shellcode
GNU gdb (Ubuntu 7.11.1-0ubuntu1~16.5) 7.11.1
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "i686-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./exec_shellcode...(no debugging symbols found)...done.
gdb-peda$ 

```

Now that the binary is loaded within `gdb`, I will create a break point and run the code. Once the `gdb` hits the breakpoint, it will stop the binary and dump the memory contents.

```sh
gdb-peda$ b *&buf
Breakpoint 1 at 0x804a040
gdb-peda$ 
```

Now let's run the code:

```sh
[----------------------------------registers-----------------------------------]
EAX: 0x804a040 --> 0x99580b6a 
EBX: 0x0 
ECX: 0x7fffffea 
EDX: 0xb7fba870 --> 0x0 
ESI: 0xb7fb9000 --> 0x1b1db0 
EDI: 0xb7fb9000 --> 0x1b1db0 
EBP: 0xbffff028 --> 0x0 
ESP: 0xbffff00c --> 0x8048479 (<main+62>:	mov    eax,0x0)
EIP: 0x804a040 --> 0x99580b6a
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a03a:	add    BYTE PTR [eax],al
   0x804a03c:	add    BYTE PTR [eax],al
   0x804a03e:	add    BYTE PTR [eax],al
=> 0x804a040 <code>:	push   0xb
   0x804a042 <code+2>:	pop    eax
   0x804a043 <code+3>:	cdq    
   0x804a044 <code+4>:	push   edx
   0x804a045 <code+5>:	pushw  0x632d
[------------------------------------stack-------------------------------------]
0000| 0xbffff00c --> 0x8048479 (<main+62>:	mov    eax,0x0)
0004| 0xbffff010 --> 0x1 
0008| 0xbffff014 --> 0xbffff0d4 --> 0xbffff2ac ("/home/anubis/SLAE/Assignment_5/exec_shellcode")
0012| 0xbffff018 --> 0xbffff0dc --> 0xbffff2da ("XDG_VTNR=7")
0016| 0xbffff01c --> 0x804a040 --> 0x99580b6a 
0020| 0xbffff020 --> 0xb7fb93dc --> 0xb7fba1e0 --> 0x0 
0024| 0xbffff024 --> 0xbffff040 --> 0x1 
0028| 0xbffff028 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804a040 in code ()
gdb-peda$ disassemble 
Dump of assembler code for function code:
=> 0x0804a040 <+0>:	push   0xb
   0x0804a042 <+2>:	pop    eax
   0x0804a043 <+3>:	cdq    
   0x0804a044 <+4>:	push   edx
   0x0804a045 <+5>:	pushw  0x632d
   0x0804a049 <+9>:	mov    edi,esp
   0x0804a04b <+11>:	push   0x68732f
   0x0804a050 <+16>:	push   0x6e69622f
   0x0804a055 <+21>:	mov    ebx,esp
   0x0804a057 <+23>:	push   edx
   0x0804a058 <+24>:	call   0x804a060 <code+32>
   0x0804a05d <+29>:	imul   esp,DWORD PTR [eax+eax*1+0x57],0xcde18953
   0x0804a065 <+37>:	add    BYTE PTR [eax],0x0
End of assembler dump.
gdb-peda$ 
```

As we can see, the first instruction loads the value `0xb` (11) onto the stack, and then the next instruction `pop`s it into `EAX`. The syscall value for `execve` according to the file `/usr/include/i386-linux-gnu/asm/unistd_32.h` file. 

So we can deduce that this will be the main part of the shellcode that will execute the command we need.

It goes through and loads `/bin/sh` onto the stack, and then sets the stack pointer into the `EBX` variable. Within this disassembly, it's not clear when it will execute, so I will issue the command `stepuntil int` which will continue the binary execution until the instruction `int` is hit.


```sh
gdb-peda$ stepuntil int
Stepping through, Ctrl-C to stop...
```

```sh
[----------------------------------registers-----------------------------------]
EAX: 0xb ('\x0b')
EBX: 0xbfffeffe ("/bin/sh")
ECX: 0xbfffefee --> 0xbfffeffe ("/bin/sh")
EDX: 0x0 
ESI: 0xb7fb9000 --> 0x1b1db0 
EDI: 0xbffff006 --> 0x632d ('-c')
EBP: 0xbffff028 --> 0x0 
ESP: 0xbfffefee --> 0xbfffeffe ("/bin/sh")
EIP: 0x804a064 --> 0x80cd
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a05c <code+28>:	add    BYTE PTR [ecx+0x64],ch
   0x804a05f <code+31>:	add    BYTE PTR [edi+0x53],dl
   0x804a062 <code+34>:	mov    ecx,esp
=> 0x804a064 <code+36>:	int    0x80
   0x804a066 <code+38>:	add    BYTE PTR [eax],al
   0x804a068:	add    BYTE PTR [eax],al
   0x804a06a:	add    BYTE PTR [eax],al
   0x804a06c:	add    BYTE PTR [eax],al
[------------------------------------stack-------------------------------------]
0000| 0xbfffefee --> 0xbfffeffe ("/bin/sh")
0004| 0xbfffeff2 --> 0xbffff006 --> 0x632d ('-c')
0008| 0xbfffeff6 --> 0x804a05d --> 0x57006469 ('id')
0012| 0xbfffeffa --> 0x0 
0016| 0xbfffeffe ("/bin/sh")
0020| 0xbffff002 --> 0x68732f ('/sh')
0024| 0xbffff006 --> 0x632d ('-c')
0028| 0xbffff00a --> 0x84790000 
[------------------------------------------------------------------------------]
gdb-peda$
```
As you can see above, the debugger stopped at the instruction `int 0x80` and the `EAX` register is loaded with the `execve` syscall value. You can also see that the stack is loaded with what our payload will execute. It will execute the command `/bin/sh -c id`. If we continue the debugger from here, we will see the output of the `id` command.


```sh
gdb-peda$ c
Continuing.
process 2849 is executing new program: /bin/dash
[New process 2853]
process 2853 is executing new program: /usr/bin/id
[Thread debugging using libthread_db enabled]
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
[Inferior 2 (process 2853) exited normally]
gdb-peda$ 
```

It's difficult to see the exact output, but if we compare the output of just running the shellcode binary we can see how it executed.

``sh
anubis@ubuntu:~/SLAE/Assignment_5$ ./exec_shellcode 
Shellcode Length:  15
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
anubis@ubuntu:~/SLAE/Assignment_5$
```

As you can see, it runs the `id` command and provides the output! Perfect, we now understand how this shellcode is working. Let's move onto the next one.


chmod Payload
-------------

We will be going through the same steps as the last payload, but in a little less depth, since this is more or less the same process.

First we will look at the options that are required for this payload.

```sh
Options for payload/linux/x86/chmod:
=========================


       Name: Linux Chmod
     Module: payload/linux/x86/chmod
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 36
       Rank: Normal

Provided by:
    kris katterjohn <katterjohn@gmail.com>

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FILE  /etc/shadow      yes       Filename to chmod
MODE  0666             yes       File mode (octal)
```

So this has two options we have to provide. The file that we want to alter, and the `chmoe` mode we want to make it. I will create a file within `/home/anubis/SLAE/Assignment_5` directory, and chmod it to be an executable.

So I created the file, and here are it's permissions:

```sh
-rw-rw-r-- 1 anubis anubis 0 Mar  9 21:52 test.txt
```
Now let's set up the payload.

```sh
sudo msfvenom -p linux/x86/chmod FILE=/home/anubis/SLAE/Assignments_5/test.txt MODE=0777 -f c
```

Note how similar this looks to the last one, but now we fed `msfvenom` our test file, and the mode we want to set the file.

We now put that into our C code shell script seen below:

```c
#include <stdio.h>
#include <string.h>

unsigned char buf[] = 
"\x99\x6a\x0f\x58\x52\xe8\x28\x00\x00\x00\x2f\x68\x6f\x6d\x65\x2f\x61\x6e\x75\x62\x69\x73\x2f\x53\x4c\x41\x45\x2f\x41\x73\x73\x69\x67\x6e\x6d\x65\x6e\x74\x5f\x35\x2f\x74\x65\x73\x74\x2e\x74\x78\x74\x00\x5b\x68\xff\x01\x00\x00\x59\xcd\x80\x6a\x01\x58\xcd\x80";



main () {


        printf("Shellcode Length:  %d\n", strlen(buf));
        int (*ret)() = (int(*)())buf;
        ret();

}
```

Now we compile it and look at it with gdb. We will set the same break point at `buf` and disassemble after hitting the break point.

```sh
gdb-peda$ disassemble 
Dump of assembler code for function code:
=> 0x0804a040 <+0>:	cdq    
   0x0804a041 <+1>:	push   0xf
   0x0804a043 <+3>:	pop    eax
   0x0804a044 <+4>:	push   edx
   0x0804a045 <+5>:	call   0x804a072 <code+50>
   0x0804a04a <+10>:	das    
   0x0804a04b <+11>:	push   0x2f656d6f
   0x0804a050 <+16>:	popa   
   0x0804a051 <+17>:	outs   dx,BYTE PTR ds:[esi]
   0x0804a052 <+18>:	jne    0x804a0b6
   0x0804a054 <+20>:	imul   esi,DWORD PTR [ebx+0x2f],0x45414c53
   0x0804a05b <+27>:	das    
   0x0804a05c <+28>:	inc    ecx
   0x0804a05d <+29>:	jae    0x804a0d2
   0x0804a05f <+31>:	imul   esp,DWORD PTR [edi+0x6e],0x746e656d
   0x0804a066 <+38>:	pop    edi
   0x0804a067 <+39>:	xor    eax,0x7365742f
   0x0804a06c <+44>:	je     0x804a09c
   0x0804a06e <+46>:	je     0x804a0e8
   0x0804a070 <+48>:	je     0x804a072 <code+50>
   0x0804a072 <+50>:	pop    ebx
   0x0804a073 <+51>:	push   0x1ff
   0x0804a078 <+56>:	pop    ecx
   0x0804a079 <+57>:	int    0x80
   0x0804a07b <+59>:	push   0x1
   0x0804a07d <+61>:	pop    eax
   0x0804a07e <+62>:	int    0x80
   0x0804a080 <+64>:	add    BYTE PTR [eax],al
End of assembler dump.
gdb-peda$ 
```

We can see very similar attributes as the first payload. The first instruction basically just extends the length of `EAX` into `EDX` so it zeroes it out. Then following instruction loads the `chmod` syscall value onto the stack, and the next instruction puts it into `EAX`. This value translates to `15` which is what the `unistd_32.h` file defines `chmod` as. Perfect.

Let's make a break point at the address of the `call` function and see what happens at that point.

``sh
gdb-peda$ b *0x804a072
```

```sh
[----------------------------------registers-----------------------------------]
EAX: 0xf 
EBX: 0x0 
ECX: 0x7fffffeb 
EDX: 0x0 
ESI: 0xb7fb9000 --> 0x1b1db0 
EDI: 0xb7fb9000 --> 0x1b1db0 
EBP: 0xbffff028 --> 0x0 
ESP: 0xbffff004 --> 0x804a04a ("/home/anubis/SLAE/Assignment_5/test.txt")
EIP: 0x804a072 --> 0x1ff685b
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a06c <code+44>:	je     0x804a09c
   0x804a06e <code+46>:	je     0x804a0e8
   0x804a070 <code+48>:	je     0x804a072 <code+50>
=> 0x804a072 <code+50>:	pop    ebx
   0x804a073 <code+51>:	push   0x1ff
   0x804a078 <code+56>:	pop    ecx
   0x804a079 <code+57>:	int    0x80
   0x804a07b <code+59>:	push   0x1
[------------------------------------stack-------------------------------------]
0000| 0xbffff004 --> 0x804a04a ("/home/anubis/SLAE/Assignment_5/test.txt")
0004| 0xbffff008 --> 0x0 
0008| 0xbffff00c --> 0x8048479 (<main+62>:	mov    eax,0x0)
0012| 0xbffff010 --> 0x1 
0016| 0xbffff014 --> 0xbffff0d4 --> 0xbffff2aa ("/home/anubis/SLAE/Assignment_5/chmod_shellcode")
0020| 0xbffff018 --> 0xbffff0dc --> 0xbffff2d9 ("XDG_VTNR=7")
0024| 0xbffff01c --> 0x804a040 --> 0x580f6a99 
0028| 0xbffff020 --> 0xb7fb93dc --> 0xb7fba1e0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 2, 0x0804a072 in code ()
gdb-peda$ 
```

Let's quickly step through what's happening. So after hitting our second break point, we see that the `FILE` parameter we fed our payload is on top of the stack. The next instruction will then put that file into `EBX`.

The next value of `0x1ff` is the hex value of `0777`, which is the `MODE` parameter we fed the payload. So once that is `POP`ed into `EBX`, we then execute the interrupt to chmod the file. Let's run the shellcode on it's own and see the affect.


Before:

```sh
-rw-rw-r-- 1 anubis anubis 0 Mar  9 21:52 test.txt
```

After:
```sh
anubis@ubuntu:~/SLAE/Assignment_5$ ./chmod_shellcode 
Shellcode Length:  7
anubis@ubuntu:~/SLAE/Assignment_5$ ls -al test.txt 
-rwxrwxrwx 1 anubis anubis 0 Mar  9 21:52 test.txt
anubis@ubuntu:~/SLAE/Assignment_5$
```
As you can see, the permissions totally changed and it is now an executable! Great, another payload understood and executed. One more to go.


Read File Payload
-----------------

Again, this will be a shorter one since it follows the same basic principles. First we will view the options for this payload.

```sh
Options for payload/linux/x86/read_file:
=========================


       Name: Linux Read File
     Module: payload/linux/x86/read_file
   Platform: Linux
       Arch: x86
Needs Admin: No
 Total size: 62
       Rank: Normal

Provided by:
    hal

Basic options:
Name  Current Setting  Required  Description
----  ---------------  --------  -----------
FD    1                yes       The file descriptor to write output to
PATH                   yes       The file path to read

Description:
  Read up to 4096 bytes from the local file system and write it back 
  out to the specified file descriptor
```

So for this payload, you provide a file descriptor (STDIN(0), STDOUT(1), or STDERR(2) ) and then a file to read, and it will send the contents of that file to that file descriptor. It defaults to the vale of `1` since that is STDOUT, which will just print the contents of the file to the screen.

Now, let's construct our payload.

```sh
sudo msfvenom -p linux/x86/read_file PATH=/tmp/SLAE.txt -f c
```

We will create this file in `/tmp/` and put inside the file `It worked!`, to show that it worked!

We then compile it the same way as the others, and load it up into gdb.

We will set up our usual break points as well, and examine the stack.


```sh
[----------------------------------registers-----------------------------------]
EAX: 0x804a040 --> 0x5b836eb 
EBX: 0x0 
ECX: 0x7fffffeb 
EDX: 0xb7fba870 --> 0x0 
ESI: 0xb7fb9000 --> 0x1b1db0 
EDI: 0xb7fb9000 --> 0x1b1db0 
EBP: 0xbffff028 --> 0x0 
ESP: 0xbffff00c --> 0x8048479 (<main+62>:	mov    eax,0x0)
EIP: 0x804a040 --> 0x5b836eb
EFLAGS: 0x282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x804a03a:	add    BYTE PTR [eax],al
   0x804a03c:	add    BYTE PTR [eax],al
   0x804a03e:	add    BYTE PTR [eax],al
=> 0x804a040 <code>:	jmp    0x804a078 <code+56>
 | 0x804a042 <code+2>:	mov    eax,0x5
 | 0x804a047 <code+7>:	pop    ebx
 | 0x804a048 <code+8>:	xor    ecx,ecx
 | 0x804a04a <code+10>:	int    0x80
 |->   0x804a078 <code+56>:	call   0x804a042 <code+2>
       0x804a07d <code+61>:	das
       0x804a07e <code+62>:	je     0x804a0ed
       0x804a080 <code+64>:	jo     0x804a0b1
                                                                  JUMP is taken
[------------------------------------stack-------------------------------------]
0000| 0xbffff00c --> 0x8048479 (<main+62>:	mov    eax,0x0)
0004| 0xbffff010 --> 0x1 
0008| 0xbffff014 --> 0xbffff0d4 --> 0xbffff2ac ("/home/anubis/SLAE/Assignment_5/read_shellcode")
0012| 0xbffff018 --> 0xbffff0dc --> 0xbffff2da ("XDG_VTNR=7")
0016| 0xbffff01c --> 0x804a040 --> 0x5b836eb 
0020| 0xbffff020 --> 0xb7fb93dc --> 0xb7fba1e0 --> 0x0 
0024| 0xbffff024 --> 0xbffff040 --> 0x1 
0028| 0xbffff028 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x0804a040 in code ()
gdb-peda$ 
```

It appears we will need to step through a bit to get anywhere good.

When we step into the `JMP`, we land at a `CALL` instruction. `call   0x804a042 <code+2>`. Let's set a break point at that address and see where we land.

```sh
gdb-peda$ disassemble 
Dump of assembler code for function code:
   0x0804a040 <+0>:	jmp    0x804a078 <code+56>
=> 0x0804a042 <+2>:	mov    eax,0x5
   0x0804a047 <+7>:	pop    ebx
   0x0804a048 <+8>:	xor    ecx,ecx
   0x0804a04a <+10>:	int    0x80
   0x0804a04c <+12>:	mov    ebx,eax
   0x0804a04e <+14>:	mov    eax,0x3
   0x0804a053 <+19>:	mov    edi,esp
   0x0804a055 <+21>:	mov    ecx,edi
   0x0804a057 <+23>:	mov    edx,0x1000
   0x0804a05c <+28>:	int    0x80
   0x0804a05e <+30>:	mov    edx,eax
   0x0804a060 <+32>:	mov    eax,0x4
   0x0804a065 <+37>:	mov    ebx,0x1
   0x0804a06a <+42>:	int    0x80
   0x0804a06c <+44>:	mov    eax,0x1
   0x0804a071 <+49>:	mov    ebx,0x0
   0x0804a076 <+54>:	int    0x80
```

This looks like something that we can figure out. So the first instruction moves the value `5` into `EAX`. This is the syscall value for `open`, which is needed to eventually read the file we gave it. It is worth noting that `EBX` is then loaded with the address that containes the location of our file, `EBX: 0x804a07d ("/tmp/SLAE.txt")`. Then finally, we have the interrupt that executes the `open` syscall on the file.

Moving along, we then load the value `3` into `EAX`, which is the syscall value of `read`, which we obviously need to read the file we want.

We put a pointer to the top of the stack into `EDI`, and move that into `ECX` as well. Finally, `EDX` is loaded with the hex `0x1000` which translates to `4096`. This is worth noting because, in the description of the payload, it mentioned that it can read a max of `4096` bytes of any file.

We then execute this syscall as well. Then, syscall value of `4` is loaded, which is the `write` syscall, since the binary needs to "write" the output somewhere. This will be `STDOUT`, which has the value `1`, and that value is loaded into `EBX` and the syscall is executed.

Finally, the `exit` syscall is loaded and executed to have the binary exit gracefully.

Let's see this binary run on it's own now below:


```sh
anubis@ubuntu:~/SLAE/Assignment_5$ ./read_shellcode 
Shellcode Length:  4
It worked!
anubis@ubuntu:~/SLAE/Assignment_5$ cat /tmp/SLAE.txt 
It worked!
anubis@ubuntu:~/SLAE/Assignment_5$
```

Perfect! Works just as expected! We know understand 3 different payloads and exactly how they function within the operating system!

<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE](https://github.com/AnubisSec/SLAE) for all the source files mentioned on this site.
