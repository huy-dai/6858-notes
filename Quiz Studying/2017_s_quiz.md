I. T/F

A. F. Decrypted when in use.
B. T.
C. F. Data is decrypted when running in applications.
D. F. This is situation when data is not at rest.

A. T. 
B. F. Hash function should always be strong, so doing it on the 2FA device vs CPU doesn't matter
C. F. Having a short cycle on the keys usability makes the # of digits irrelevant 
D. T. OS kernel can lead to exploits in other programs.

ABCD not useful

ABCD not useful

II Buffer Overflows

5. Exploit: 
6. 
```py
"A"*16 
+ struct.pack("<Q",0x40a01234) 
+ struct.pack("<Q",0x7fff0114)
+ struct.pack("<Q","x.txt".encode('utf-8'))
```
On the correct path but not quite correct.
```text
"x.txt" + \0 + "A"*10 + 0x40a01234 + 0x7fff0114 + 0x7fff0100
```
I notice they don't really care about the syntax and more just what kind of values you were putting down, which makes sense.

III. Baggy Bounds

6. Total size is 32, which means there's 32-24 = 12 bytes of padding. `0x10203040 + 0d12 + 0d18 = 0x1020305e`. Note we are still within the range.

7. High bit indicates OOB. True address is 0x1001005a. We see that the expected size for struct is 24, which would cause it be formatted region of 32 bytes (e.g. 32 bytes aligned). We see `0x1001005a % 32 = 26`, so it is underflowed 6 bytes from start of struct. Thus we are 0d12 + 0d6 = 18 bytes behind original location, so `N = -18`

IV. OKWS / Lab 2

8. Have Zookd be the main service reading requests from client sockets, then let it dispatch to the corresponding service only, read the response, ensure that its Content-Length is correct, and send forwards it to the user. All manipulation with TCP socket is done at zookd.

V. Trusted Hardware (Skipped)

VI. Native Client (Skipped)

VII. Capsicum (Skipped)

