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