# OS and VM Isolation

## Prereading

Paper question: After reading the paper, I'm still a little bit confused about the security of Firecracker, since they did not provide much or any tests regarding security in the paper (only performance). Is it secure mainly because Firecrackers draws / shares codebase from well-designed container platforms like KVM and QEMU?

**Answer:** The main security metric is that Firecracker was written in Rust (so,
no buffer overflows), has relatively very few lines of code (50KLOC),
and runs in a really restrictive jail (section 3.4.1).

## Case Study: AWS Lambda Service

AWS Lambda allows Amazon to run customer's code / application in a Linux environment in isolated containers. Customer provides application binaries that Amazon needs to run. Amazon in turn run those services, usually across different physical machines.

## Isolation Approaches

* Option 1: Run applications side-by-side in the same Linux machine
  * Linux certainly can handle multiple processes
  * With processes, you can set each to run under different UserIDs. With this, you can use the userids to have **fine-grained control** over which application sees what file. However, this also makes it harder to manage, especially since you need to set it for all files, and there are also many shared resources in the Linux system. Also there's no isolation of network stack.
  * To help with the filesystem permissions, we can use **chroot**. Chroot associates a root/ jail directory for each process, and for unprivileged processes they are bound within that directory. 
* Option 2: Containers
  * Containers rely on *namespaces*, among other things, for isolation
  * **Namespaces** provide a scope for *all* resources. Provide virtual filesystem namespace, network namespace, process pid namespace, etc. Namespaces allow us to have coarse grain control over what resources each group has access to, and also help isolate groups from each other
    * Namespaces depend on *cgroups* to control resource use
  * A **container** contains a full Filesystem image running its own processes and applications. 
    * One non-security benefit of container is that we don't have to worry about dependency issues between different applications (if they are in different containers).
    * Security wise, the inherent isolation between containers means that it's not easy compared to the processes isolation to mess up configuration and expose large parts of your system unintentionally.
* Option 3: VM
  * Each VM runs its own application.
  * Potentially provide more isolation than even containers, since there is less attack surface (a VM exposes fewer operations of the underlying host OS)
    * Bugs in Virtual Machine which allows a process to escape is much rarer than those in containers.
  * Drawbacks (from perspective of Lambda AWS developers): 
    * Slower start time, VMs are more resource intensive (since you can still only have one VM per application) - more overhead, and VMM also have their own share of bugs [at least more than they would like for their service]
    * **Also need hardware support**. For example, only x86 servers can run x86 VMs, and ARM applications will require its own set of ARM servers.
* Option 4: Language-based isolation
  * To be discussed in later lectures

## Why did Firecracker not just use standard Linux containers?

- Linux vulnerabilities
  - They worry about attacks on the Linux operating system that might invalidate the security guarantees of the containers. Also there might be bugs in the Linux kernel that allows a program to break out of the container.
  - In terms of the **attack surface**, Linux kernel has a lot of places where it can be attacked. Over 200 syscalls, 1000s of IOPS operations, and lots of shared state ("resources") in the kernel (e.g. **monolithic kernel**). The kernel is also made over 10M lines of code, most of which are written in C code.
    - One notable functionality in Linux is that we can use **seccomp-bpf**, which is another built-in part of the kernel that allows us to put in a filter that specifies which syscalls are allowed to run within the process. This filter is also inherited by any forked child process. 
    - However, the paper complains that this is one "annoying" payoff since they don't know for sure what syscalls that their customer actually needs on a program-to-program basis.
  - Thus the team that made Firecracker decided to work on their own implementation

## What's involved in implementing support for VMs?

- Virtualizing memory
- Virtualizing CPU
  - Dealing with nested page tables, interpreting instructions, and also dealing with managing registers that typically only the kernel has access to
  - This is usually done with hardware support in modern processors.
- Virtualizing devices
  - Allowing creation of virtual disks + filesystems, USB devices, GPU / graphics, network interface, etc.

## Linux KVM

Layout:

```text
---------------------------
VM (containing virtual memory)
---------------------------
Linux Kernel | KVM Process
```

**KVM** essentially converts Linux into a type-1 hypervisor. KVM has a component/ process which runs at the level of the Linux kernel, which allows for emulating a processor and executing commands more directly on hardware. The KVM process is also responsible for managing page tables (utilizes hardware support).

If the VM hits something which the hardware doesn't know how to emulate, the command gets sent to kernel / KVM, and is directly sent to a "helper", which is usually QEMU. QEMU provides additional support for commands that isn't recognized by the kernel.

**QEMU** implements virtual devices, similar to what real hardware would have. It also implements purely-virtual devices (*virtio*), for example, nested virtual machines. QEMU also provides some BIOS implementation to start running the VM.

Fun fact: QEMU actually existed before KVM was created. KVM was designed in a way that allowed it to take advantage of pre-written emulation features of QEMU

## Firecracker Design

Firecracker still uses KVM, but they *re-implement* QEMU. They really focused on limiting the list of supported devices - only support virtio network, virtio block (disk), keyboard, and serial. 

For example, their version supports virtual disk but not virtual filesystem device (this, for example, can be used to share filesystem between host OS and guest OS). The reason they can limit this list down is because the Lambda AWS is not allowing their users to run arbitrary VMs, e.g. only Linux applications.

They also rewrite a lot of **QEMU** in Rust, which is a more memory-safe language. Their codebase is also much smaller in virtue of their minimization; only 50K lines. Thus there is much less likely for buffer overflow.

Firecracker also do not support instruction emulation, except for only necessary instructions like CPUID, VMCALL/VMEXIT, etc.

In addition to these changes, Firecracker also uses various container strategies like:
- chroot
- namespaes
- userids
- seccomp-bpf filter

All of these work together to say in that in the case that bugs in VMM are exploited, it is harder to escalate the attack.

## How is Firecracker implemented for Lambda AWS service?

There is a front end that is responsible for taking in and dispatching service requests. The actual creation and management of containers are done by the Virtual Manager.  In the backend there is the actual physical servers which hosts the containers.

In addition they also uses a **specialized kernel** in the MicroVMs that boots much quicker (100 msec). They also use an optimization where they would pre-boot VMs before customer code comes in, which helps reduce amortized latency.

## Evaluation of Firecracker

The main two good aspects of Firecracker is that it has very little overhead and high security.

Performance seems to be acceptable / OK. CPU performance is basically KVM, though the device I/O performance is not so great. For example, speeding up virtual disks perf will need concurrency while virtual networks can benefit from PCI pass-through.

However, in the security aspect, Rust is much less bug prone, the VMM process is jailed, and much less code is in the VMM. Still, KVM bugs can still undermined Firecracker's isolation (certain cases has been documented in the past, like bug of making VM memory grow forever from spamming serial port, or like improper management of memory by KVM that weakened security implementation of Firecracker's QEMU)

