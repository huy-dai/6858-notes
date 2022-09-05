Side-channel attacks and Spectre 
================================

Side channel attacks: historically worried about EM signals leaking.
  [ Ref: http://cryptome.org/nsa-tempest.pdf ]
  Broadly, systems may need to worry about many unexpected ways in which
    information can be revealed.

Many information leaks have been looked at to extract crypto keys
  How long it takes to decrypt.
  How decryption affects shared resources (cache, TLB, branch predictor).
  Emissions from the CPU itself (RF, audio, power consumption, etc).

Many side-channel attacks are not crypto-related.
  Operation time relates to which character of password was incorrect (see Tenex below)
  Or time related to how many common friends you + some user have on Facebook.
    If attacker want to find out how many shared friends
  Or how long it takes to load a page in browser (depends if it was cached).
    If attacker wants to find out if someone looked a particular page
  Or recovering printed text based on sound from dot-matrix printer.
    [ Ref: https://www.usenix.org/conference/usenixsecurity10/acoustic-side-channel-attacks-printers ]
  Or network traffic timing / analysis attacks.
    Even when data is encrypted, its ciphertext size remains ~same as plaintext.
    Recent papers show can infer a lot about SSL/VPN traffic by sizes, timing.
    E.g., Fidelity lets customers manage stocks through an SSL web site.
      Web site displays some kind of pie chart image for each stock.
      User's browser requests images for all of the user's stocks.
      Adversary can enumerate all stock pie chart images, knows sizes.
      Can tell what stocks a user has, based on sizes of data transfers.
    Similar to CRIME attack mentioned in Paul Youn's lecture earlier this term.
    Or https://blog.appcanary.com/2016/encrypt-or-compress.html
  But attacks on passwords or keys are usually the most damaging.

Famous password timing attack (Tenex operating system)
  Time page-faults for password guessing.
  Suppose the kernel provides a system call to check user's password.
    Checks the password one byte at a time, returns error when finds mismatch.
    (Tenex stored the password as plaintext.)
  Adversary aligns password, so that first byte is at the end of a page,
    rest of password is on next page.
  Somehow arrange for the second page to be swapped out to disk.
    Or just unmap the next page entirely (using equivalent of mmap).
  Measure time to return an error when guessing password.
    If it took a long time, kernel had to read in the second page from disk.
    [ Or, if unmapped, if crashed, then kernel tried to read second page. ]
    Means first character was right!
  Can guess an N-character password in 256*N tries, rather than 256^N.
  [ Ref: http://xeroxalto.computerhistory.org/_cd8_/pup/.plfsup.mac!2.html ]
    Not quite the original code, but similar: [chkpsw] calls [stcmp] to compare.

Cache timing attacks: dm-crypt in Linux.
  [ Ref: https://www.cs.tau.ac.il/~tromer                 /papers/cache-joc-20090619.pdf ]

What's new with Spectre and Meltdown attacks?
  Caused people to adjust their threat model
    Side channels were assumed away
    Now micro-architectural side channels are real
    $ grep . /sys/devices/system/cpu/vulnerabilities/*
  Exploit using speculative execution
    Standard processor feature to get high performance
  Adversary has some control over speculative execution
    Big difference from earlier cache timing attacks
  Speculative execution has observable side effects
    Brings values into cache, even on miss
  Breaks process isolation
    Attacker probes cache
    High bandwidth side channel
  Architectural side channel
    Not easy to address
    Several workarounds
  Danger: exploitable remotely through Javascript, WebAssembly, ..

Processor features exploited
  Both features to obtain high performance
  Speculative execution
    if cond {
       ...
    }
    Processors guess branch and speculative execute instructions
    On today's processor it may be many instructions
    On failed speculation clean up processor state
      registers, etc.
  Caching
    Processors have several levels of caches
      Last level is shared
    On failed speculation, data stays in cache

Many attack variants
  1. Bound check bypass (Spectre)
  2. Branch target injection (Spectre)
  3. Data access bypasses protection checks (Meltdown)
  4. ... any other micro-architectural side channel ...
  Surprisingly long list of attacks.
    E.g., see https://dl.acm.org/doi/pdf/10.1145/3492321.3519559

  First look at variant 3, and then come back to variants 1+2

Outline of exploit of variant 1
  Kernel has a secret byte
    E.g., offset for Address Space Randomizaton
  Kernel has a code fragment ("gadget") as follows
    if (offset_from_attacker < sz) {   // attacker arrange miss on sz
      // trains the branch predictor to speculatively executes this branch
      // arrange that array1 is in cache so that the following
      // instruction runs quickly and there is time to start next one
      v = array1[offset_from_attacker]  // read secret byte
      // attacker cannot read v, but arranges array2 is not in the cache and uses v
      v1 = array2[v]                   
    }
  Once speculation fails, processor squashes internal processor state
    But doesn't remove array2[v] from cache
    External to the processor
  Attacker measures time to read of
    array2[0]
    array2[1]
    if one them is fast, then attacker knows value of v

Concrete example: appendix A
  [ lec/spectre.c ]
  [ show code, compile wo optimizations, and run ]
  Illustrative
    Not a real attack
    Victim and attacker code in same address space
  Victim code
    Gadget is present
    Secret
    why unused1? unused2? (force size to be in different cache line from array1)
  Main function
    Try to read secret a byte at the time
    Using readMemoryByte
      malicious_x is the offset_from_attacker
    Uses cache measurements to read byte
  ReadMemoryByte
    Does 1,000 tries -- why?
      But breaks earlier
    Flush caches
      why 256?  (a byte has 256 possible values)
      why 512?  (prefetch?)
    Train branch predictor
      Why complicated way of computing x
      Invoke victim's gadget
    Time every possible value
      Count cache hits
  Bandwidth of side channel
    10KB/s

To make attack real:
  Find gadget in victim process
  Attacker process must be able to access array2
  Evict sz in victim process
  Evict array2 in victim process
  Train predictor in victim process

"Real" attack: steal secret from victim process on Windows
  Exploit the presence of DLL (dynamically-loaded library)
  Mapped in each process address space
    Which can be read and clflushed'd by any process
    Can even be written

"Real" attack: steal secret from Linux kernel
  [Ref: https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html]
  Exploit eBFP in kernel
  Two filters:
    1. to train BP and find address of "array2"
    2. to leak secret data
  Measure array2 from user space
    User space is mapped in kernel address space

Discussion
  Attack breaks process isolation
    Serious attack!
  Attacker can be remote
    Attack can be performed from JavaScript
  Any micro-architecural state maybe a side channel
    Branch predictor, write-buffer, cache-coherence directory, ...

Variant 2
  Kernel code:
    if (off < sz) {
        v = array1[off] 
        (*f)()  // an f that read memory depending on v
    }
  Processor may speculative executes f
    important for object-oriented code
  Attacker chooses convenient f in kernel
    Like a return-to-libc attack
  Requires reverse engineering of branch predictor
    To train it to invoke the chosen f
  
Variant 3
  User-space code:
    mov rax, [some_kernel_address]
    and rax, 1
    mov rbx, [rax+some_user_mode_address]
  First instruction is several micro operations
    check and load
  Processor execute instructions out of order
    first load, then check
    memory isn't committed, but address is in cache
    use flush+reload probe to measure if address is in cache
  Works because one address space with kernel and use
    Kernel-code is present in user process's page table
    Kernel code is mapped not readable by user code

Fairly real attack using "variant 3": stealing keys from SGX enclaves
  See https://www.usenix.org/system/files/conference/usenixsecurity18/sec18-van_bulck.pdf

Mitigations.
  Variant 1.
    Linux kernel: Code analysis to avoid vulnerable code
      modify code to constraint index in gadget code
      https://lwn.net/Articles/752408/
      include/linux/nospec.h
      For example:
	 if (index < ARRAY_SIZE)
	     // array_index_nospec() blocks speculation
	     index = array_index_nospec(index, ARRAY_SIZE);
	     return array[index];
	 }
    Browsers: avoid generating vulnerable code
      https://v8.dev/blog/spec
  Variant 2. Microcode update and kernel patches (e.g., retpoline)
    a return trampoline (https://support.google.com/faqs/answer/7625886)
      has an infinite loop that is never executed
      prevents the CPU from speculating on the target of an indirect jump
    slows down all indirect jumps
      but another plan (optpoline) resolves indirect jumps at runtime
      and replaces them with direct jumps
      https://lwn.net/Articles/774743/
  Variant 3. Kernel page table isolation (KPTI)
    [Ref: https://en.wikipedia.org/wiki/Kernel_page-table_isolation]
    Developed before Meltdown was known
    Two page tables: one for kernel and one for user
  Harden browser
    No precise timing info
  Performance overheads
    See https://dl.acm.org/doi/pdf/10.1145/3492321.3519559

Other observable side effects of speculative execution
  Power consumption
  Bus traffic
  ...

Other types of timing attacks.
  Cache analysis attacks: processor's cache shared by all processes.
    E.g.: accessing one of the sliding-window multiples brings it in cache.
    Necessarily evicts something else in the cache.
    Malicious process could fill cache with large array, watch what's evicted.
    Guess parts of exponent (d) based on offsets being evicted.
  Cache attacks are potentially problematic with "mobile code".
    WASM modules, Javascript, etc running on your desktop or phone
    Mobile code is untrusted, and may come from an attacker
    It is sandboxed, but it still may be able to measure side channels

How to stop side channels?
  Remove side channels
    E.g., redesign processor to not commit to cache
    But every shared resource could be a side channel
  Reduce shared resources when processing secret data
    Process confidential data on a separate core
    With its own cache, etc.
    Requires refactoring of software --- not easy
  Measure channel and make them low bandwidth
    Hard to exploit

References:
  https://googleprojectzero.blogspot.com/2018/01/reading-privileged-memory-with-side.html
  https://cyber.wtf/2017/07/28/negative-result-reading-kernel-memory-from-user-mode/
  https://eprint.iacr.org/2013/448.pdf
  http://css.csail.mit.edu/6.858/2014/readings/ht-cache.pdf
  http://www.tau.ac.il/~tromer/papers/cache-joc-20090619.pdf
  http://www.tau.ac.il/~tromer/papers/handsoff-20140731.pdf
  http://www.cs.unc.edu/~reiter/papers/2012/CCS.pdf
  http://ed25519.cr.yp.to/
  http://cseweb.ucsd.edu/classes/fa05/cse240a/bp.pdf

Side-channel attacks on RSA
  [ Ref: https://crypto.stanford.edu/~dabo/papers/ssl-timing.pdf ]
  Victim Apache HTTPS web server using OpenSSL, has private key in memory.
    Goal: extract private key
  The basic plan is like Tenex system.
    Guess a bit at the time.
    But operations on server are noisy and so attack more complicated.
  Connected to Stanford's campus network.
  Adversary controls some client machine on campus network.
  Adversary sends specially-constructed ciphertext in msg to server.
    Server decrypts ciphertext, finds garbage padding, returns an error.
    Client measures response time to get error message.
    Uses the response time to guess bits of q.
  Overall response time is on the order of 5 msec.
    Time difference between requests can be around 10 usec.
  What causes time variations?  Optimizations to implement RSA efficiently
    Karatsuba vs normal; extra reductions.
  About 1M queries seem enough to obtain 512-bit p and q for 1024-bit key.
    p and q are the two primes that RSA uses to construct public and secret key
    Only need to guess the top 256 bits of p and q, then use another algorithm.
  Counter measures are needed (e.g., RSA blinding)

How worried should we be about RSA timing attacks?
  Relatively tricky to develop an exploit (but that's a one-time problem).
  Possible to notice attack on server (many connection requests).
    Though maybe not so easy on a busy web server cluster?
  Adversary has to be close by, in terms of network.
    Not that big of a problem for adversary.
    Can average over more queries, co-locate nearby (Amazon EC2),
      run on a nearby bot or browser, etc.
  Adversary may need to know the version, optimization flags, etc of OpenSSL.
    Is it a good idea to rely on such a defense?
    How big of an impediment is this?
  If adversary mounts attack, effects are quite bad (key leaked).

