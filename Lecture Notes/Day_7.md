## Software Fault Isolation + WebAssembly

WebAssembly is designed to be fast, portable, and safe.

## Motivation
In a browser the Javascript engine is responsible for running JS code from each website in its own isolated process. However to the developers they felt that JS is a weird and slow compilation target, especially if your web application is originally written in Haskell or C/C++ or some other language. Also we may want to be able to run usually unsupported languages like Rust or Go (and even though those are known to be memory-safe, they aren't necessarily safe when running in a browser setting).

- Why not compile to x86 code?
  -  Even though it's fast, x86 code is still not safe and it's architecture specific.
- Why not OS/VM isolation?
  - Device may not support for VM, and OS may differ in their isolation mechanisms
  - VMs and containers takes time to start up
  - Also these measures often require root privileged process to use VM and containers.

## Software Fault Isolation (SFI)

The idea is that for developers that will their application code (written in a variety of languages) and give it to an SFI-aware compiler that produces a binary. That binary is then shipped over network to the users to their browser, which then inspects and validates that binary. Lastly we will run the binary.

If the binary errors out the browser will catch it and prevent it from corrupting other processes (using "traps"). Note that this still allow memory corruption *within* the bad module, but limits the effect from expanding outwards.

## WASM

Each module contains functions, global variables, tables of function pointers (for indirect calls), import / export, and memory for heap. WebAssembly intentionally **separates out** the use of memory instead of storing everything together which allows the validator to better detect when code goes out of bounds.

The big challenge to writing WASM is to balance *performance* and *isolation*. 

To achieve security functionality + isolation we could write an interpreter for whatever code we want to run (JS works sort of along this idea), and interpreter makes sure to allow code to access outside of the isolation functionality. For example, we can just run QEMU in the browser to emulate x86. However, this approach is slow! 

To achieve better memory, instead of interpreting you want to translate to native instructions and then run on the native instructions directly. However, as mentioned earlier, we can't do x86 directly because it's really hard to validate x86 code statically because we really don't know what memory addresses will be accessed or if code will jump someplace outside of module (because these info are computed at runtime). Also we will need to deal with syscalls.

On the other hand, WASM divides a program into modules / basic blocks (through division at function boundaries). Ensures state and control flow:
 - State: Don't allow accessing anything outside of the module. We generate & add code that will perform checks of memory accesses at runtime. 
   - Note: WebAssembly employs a smart design to limit the number of checks that they have to do as to not affect performance too much 
   - We only allow accessing to:
     - Stack values: At every point in a function, it's known how man values are on the runtime stack. The control flow within a function must always agree on stack contents (e.g. the number of stack items)
     - Global variables: These are known ahead of time at compile-time. Runtime also maintains some way to access the base of the global.
     - Local variables: Every function has well-known number of local variables, so with globals, we can check the offset and make sure it is in-range of accessible range. Note that sometimes local variables are accessed many times (e.g. function called recursively), so local variables are stored on the (native) stack
     - Heap memory, grows from 0 upwards: WASM adds bound-checking on memory operation. It mainly makes sure the resulting code cannot corrupt state outside of sandbox (e.g. accessed address must be in determined heap memory region)
       - If we're checking like the first 64 indices of an array, we only need to check the 64th index address isn't beyond the boundary.
 - Control Flow: Don't allow jumping to any code that is not "properly translated". Overall this ensures **control flow integrity (CFI)** of the program. This means that we can only jump to modules that we've generated and have added checks for, and not:
   - Code outside of sandbox
   - Mis-aligned code
   - Code in sandbox which bypasses the check


## Implementation

In WASM, when we call a function *f*, we are going to create a set of local variables for *f* and an operand stack. If *f* calls *g*, we also create the set of local variables for *g* and its own operand stack.

In x86 world, all of this info is stored on the native stack. When we call *f*, we push some return address to the stack, followed by its local variables and operand stack. When *g* is called, we save the return address within *f*, and then add *g* local variables and operand stack (so like a normal assembly program essentially). When WASM programs wants to say, access some local variable within a function, it will need to add the var offset with a constant representing the length of all the "chunks" stored before it (and these are statically known). 

Additionally, unlike a a normal x86 program, the validator will be there to prevent any store operation that goes outside of your expected bounds (because WASM is helping track of more information about program modules to ensure state and control flow integrity). This is because WASM uses **type safety** to ensure there is a known stack depth at any instruction, regardless of how we got there in the code. This is a really important effect resulting from the design of WASM code!

**Important Note:** It's entirely possible for a C program compiled to WebAssembly to have buffer overflow, for example, by corrupting heap-allocated memory region. However, main job of WebAssembly is to ensure WASM code can't affect rest of system.

WASM also performs *function type checking*, since  function arguments are accessed just like local vars and WASM functions are allowed to modify some number of runtime stack location, depending on how many arguments it declares itself to take. Thus, we need to make sure the caller gives us enough arguments on the stack, otherwise function may be able to read/write other items on the runtime stack, which would be bad.

For indirect function calls, we have a table of jump targets, which the compiler ensures that all of which are valid function addresses.

## Performance Improvement

There are some ways to increase performance of WebAssembly with some OS and hardware support. The idea is to map a large reason of virtual memory that covers all of the memory that you may need for the program (in that case most of it will be unmapped). 

In WASM spec it says memory is at most 4GB in size, but virtual memory actually reserves 8GB of memory for all possible worst-cases. For a memory store operation in WASM it would be done as: `*(membase + address + offset) = value`. Since both address and offset has to be 32-bit values, it means resulting address will be at most 8GB from membase. Thus in the worst case, we will hit a page fault is access to memory is beyond allocated size, and so we can trigger an exception to the OS. This in turns allows us to have less checks in the program since this virtual memory size already prevents program overflow.

## Result / Outcome

WASM is widely supported in browsers now, and it's even uses in certain applications like Adobe Photoshop. There also has been quite a number of extensions made for WASM, with new features like adding support for threads, vector instructions, etc.


Also even stuff like AWS Lambda, Fastly, Cloudflare CDNs also use WebAssembly since it allows running code quickly on a server near requesting user.