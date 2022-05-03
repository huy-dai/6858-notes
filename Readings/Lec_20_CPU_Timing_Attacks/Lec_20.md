# Spectre and Meltdown Processor Vulnerabilities

Link to [research paper](https://css.csail.mit.edu/6.858/2022/readings/spectre-meltdown.pdf)

## Intro

In recent years there has been an uptick in the number of attacks targeting hardware vulnerabilities. Attacks like Spectre and its variants (including Meltdown) are of great concern because they can be widely available and it takes much longer for hardware to be replaced or patch after a fix is found.

### ISA Spec

Since the 60s, computer systems has defined hardware correctness through ensuring *timing-independent* functional behavior that complies with a particular **Instruction Set Architecture (ISA)** spec. The ISA provides a layer of abstraction since software is written to the architecture spec, while hardware can use **microarchitecture** to make certain optimizations and tradeoffs in actual implementation custom to intended use. 

### Out-of-order execution

To maximize **Instruction Level Parallelism**, modern out-of-order processor allow for many instructions to scheduled onto the hardware in parallel. This requires the CPU to predict the outcome of branches so it can choose a path to speculate against. Correct predictions have their results made visible, while incorrect predictions have their results discarded

## Side Channel Attacks

Spectre variants are a form of **side-channel attacks** (a type of attacks which has existed since the 70s) in which microarchitecture state, becomes observable at an architectural level by an attacker program sharing resources with the victim. This state can include secrets loaded into a shared architectural state, including data reference via speculation prior to completing access, validity, or bound checks. 

The resource through which secrets are extracted can take form of shared cache, but it can also include other shared structures like **Translation Lookaside Buffers**.

## Spectre

To exploit Spectra, a malicious attacker causes intentional misspeculation of instructions. These **transient instructions** will end up being aborted prior to retirement and thus do not have an architectural impact. However, they can cause a change to the internal microarchitectural state that can be observed using a side channel (such as a shared cache). This is done through observing cache access time to determine whether data, which have been brought into cache as a result of speculative execution, are present in a particular location.

The typical structure of an attack looks like the following:

1. Train some microarchitectural state, e.g.,
   1. Clear 256 blocks from the cache at address X
   2. Prime the branch predictor to predict a given branch "not taken"
2. Save secret in the microarchitecture (usually through a load instruction)
3. Extract secret from the microarchitecture 
   1. Time the accesses to 256 cache blocks at X, the block that hits and completes faster corresponds to the value of memory at loaded address

Spectre variants can different a different underlying hardware vulnerability, and the impact varies significantly. Some vulns can be exploited by interpreted code running in a sandbox, some can be exploited across VM boundaries, and some even allow attacks from applications on the kernel or other applications.

Software mitigations for Meltdown:

1. Preventing vulnerable processors from having valid address translations for privileged (kernel) memory when running in an unprivileged (user) state. This is known as “Page
Table Isolation” (PTI). It is the approachused on Intel x86-64 and Arm processors.
2. Preventing the Level 1 data cache from containing secret data that could be loaded by malicious user code. This is achieved through flushing the L1 data cache on
return from the OS or Hypervisor into application code. It is the approach used on IBM POWER processors.

## Where to go from here?

These vulnerabilities will not be solved by hardware and software engineers working in isolation. Software engineers will need to have some notion of how processors behave, an understanding of caches, memory management, specu-lation, and out-of-order execution. Second, Open Source specifications (ISA) and implementations can help. Security benefits from  “many eyeballs,” cleaner design with smaller
attack surfaces, and from designs that the security researchers can use to collaborate on solutions that are not specific to one commercial processor vendor. 
