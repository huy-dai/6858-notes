I. T/F

1. T
2. **F** Google doesn't have a concrete way to deal with intentional hardware-level CPU bugs. They do have vendor vetting and customized board, but it's still possible for CPU manufacturers to insert their own backdoor.
3. F
4. **T**. 2FA can help fight against key logging on the user side.
5. T. Physical security is still an important step of security.

1. T. SMS codes are vulnerable to MITM attack.
2. F
3. T. 2FA key is never exposed directly to software on user's computer.
4. **F**. Even U2F adds another step, it still can't make guarantee that the identity of the person behind computer is same as person who is trying to log in. Also the adversary may choose to steal the USB dongle.

(Skipped)

II. Buffer Overflows

4. First attack: There is a buffer overflow on line 9, where the string at *name is appended to the end of pn, which is only 1024 bytes long. Thus, we can use this to overflow the buffer and add to it our own shellcode which, say, opens a shell. We also override the return address such that at the end of http_serve() it will return to a location within the stack to execute our shellcode.

Second attack: We can provide a valid path which escapes the current directory through a series of "../../.." upward directory movement, and then narrows down to a sensitive file like /etc/shadow and /etc/passwd to leak into about the computer (uses line 20). Alternatively, we can also leak the source code of this application.

Third attack: Buffer overflow to overwrite *handler* variable with a libc function like unlink.

5. Input:
```py
cwd = '/home/httpd'
str1 = '/home/httpd/grades.txt\0'
exploit = "A"*(1024-len(cwd)) + 0xBCBCBCBC + 'A'*8 + 0xbffff5a4 + str1
```

HTTP serve has the cwd path of `/home/httpd`
Also, the interesting part here is that since the str1 is nested after the cwd, it's easier to supply the path at the end rather than in the front. We're also assuming the architecture that they are working with here is x86 which is why we don't have to worry about `pop rdi` gadgets.

III. Baggy Bounds Checking

6. Yes. Baggy bounds allocate exactly 1024 bytes for `pn`, and thus it will prevent any writing beyond the 1024 characters.

7. No. Baggy bounds will then allow strcat to overwrite up to four bytes after the end of `pn`, since the last four bytes of the allocated region will be considered padding and does not raise an error if it is written to.

(Misunderstood question) No. In that case `pn` will be allocated 2048 bytes, which means the attacker will be able to write at most 1024 characters beyond the end of `pn`. However, this operation will not affect any other variable.

IV. Capsicum (Bypassed)

V. Native Client (Bypassed)

VI. IOS

10. Alyssa can save the OS image for every version of iOS that she wants to test on her phone (thus she would need to install them in order from oldest to latest). Afterwards, she can simply erase the current OS image and install the custom version that she wants. Since each OS image she has is joined with her ECID, the system will still boot correctly. 

VII. Android

11. Ben can release a malicious app on the App Store that has the same .apk filename as a common app (like, say, GMail). If Ben can convince users to download that app, it will causes for there to be two identical ZIP files in the system. In that case, next time the user tries to open the GMail app, the system will end up loading Ben's malicious software instead.

**Note:** This attack doesn't work because the issue is not similar named ZIP files but contents within the ZIP file. 

So the attack here is to create a custom version of an existing app, make copies of existing scripts (keeping same name) but add his own's malicious code inside. When the installer checks the .apk, it will only see the good version (and so verification passes), but when it comes time to run the app the malicious version of the script would be loaded. If Ben can distribute this version as an update to the existing app, it would still download normally without alarm.