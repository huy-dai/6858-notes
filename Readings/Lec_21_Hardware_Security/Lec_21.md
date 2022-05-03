# Betrusted.io

Link to [website](https://betrusted.io/)

Betrusted is a secure and private communication systems, which uses a co-designed hardware and software solution. It gives users an evidence-based reason to trust that their private info are kept private.

## Values

1. Transparency: Betrusted uses as an FPGA that users can recompile the reference processor design from source (and with certain specialized machines that are a few thousand dollars in cost to do optical verification of circuit). Also **Functional (RTL)** transparency enables experts to find and possible fix architectural vulnerabilities.
2. Simplicity: Betrusted uses a small but powerful code base that have been narrowed down to be of a simple design but still allows for high computational capability.
3. Completeness: Provide for end-to-end security solution, beyond just chip-only trust solutions like TPMs, SGX, and secure enclaves.
4. Self-sealing: The device can generate and seal its own private keys on-chip in a transparent and open manner.

## Primary Use Cases

* Text based chats
  * Signal like texting but only chat feature
  * Has Unicode support and multi-lingual IME
* Voice Chat
  * Asynchronous messaging is MVP
  * Real-time voice call may be possible
* Banking, crypto apps, and authenticator tokens are also realistic additions

There's no browser, games, video, social media, or app store.

## Components of Betrusted

* Secure hardware: Betrusted uses the application-neutral platform called Precursor to build its applications on. It also uses a trustable screen and a real, trustable keyboard.
  * Trustable screen allows for user to find Trojan circuits through non-destructive inspection, and there's no presence of a silicon chip.
  * Trustable keyboard has its wires visually inspective, and X-ray be employed to rule out buried traces. There's also no silicon chips.
* **Xous**: Is the microkernel OS for Betrusted written in Rust. 
  * Xous uses the *message* as its fundamental data unit. memory is shared between processes as a message (kind of like Android's design) with attributes of: Borrow, Mutable Borrow, or Move.
  * The hope is that by matching OS level concepts around sharing memory to match the underlying language Rust (borrowing much of its memory borrow checking), they are able to eliminate or greatly reduce large class es of memory-safety related bugs.
* PDDB: As with any security system we must also strive for **plausible deniability (PB)** of any secrets that may or may not be contained within the system. PB is possible on traditional systems but is tricky since it is hard to train the user to use the tools.
  * PDDB takes advantage of the Message-oriented nature of Xous to entirely do way with the notion of files and volumes.
  * Instead, data is stored in a database as a set of key/value pairs that associated with a security state.
  * They blend patterns of secure and insecure data access all to the way the hardware, while keeping end-user applications largely oblivious to the entire PD process. 
  * As a result, the process of deleting a data set is identical to that of forgetting a PIN code.
