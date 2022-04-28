# Tor: The Second Generation Onion Router

Link: <https://css.csail.mit.edu/6.858/2022/readings/tor-design.pdf>

# Overview

Tor is a circuit-based low-latency anonymous communication service. More specifically, it is a distributed overlay network designed to anonymize TCP-based application.

Clients choose a path through the network and build a **circuit** in which each node, called **onion router**, knows its predecessor and successor, but no other nodes in the circuit. Traffic flows down the circuit in fixed-size *cells*, which are unwrapped by a symmetric key at each node. 

The second version adds the following characteristics:
1. perfect forward secrecy: Instead of using a single multiple encrypted data structure, Tor now uses an incremental or *telescoping* path-building design, where the initiator negotiates session key s with each successive hop in the circuit. Once the keys are deleted, subsequently compromised nodes cannot decrypt old traffic. 
2. congestion control: Tor uses a *decentralized* congestion control which uses end-to-end acks to maintain anonymity while allowing nodes at the edge of the network to detect congestion or flooding and send less data until the congestion subsides 
3. directory servers: Earlier versions flood state information through the network, which is unreliable and complex. Now, certain more trusted nodes act as **directory servers**, which provide signed directories describing known routers and their current state. Users periodically download this info via HTTP
4. integrity checking: Tor hampers MITM attack by verifying data integrity before it leaves the network (maskre sure no node in the circuit could change the content of data cells as they passed by)
5. configurable exit policies: Tor allows each node to advertise a policy describing the hosts and ports to which it will connect. It allows operation to dictate the different types of traffic they are comfortable having exit from their node.

## Inherent Drawback

Since Tor is focused on low-latency designs, it is difficult to prevent an attack who can eavesdrop both ends of the communication from correlating the timing and volume of traffic entering the anonymity network with traffic leaving it. Countermeasures have been added to the design, though most rotect primarily against traffic analysis rather than traffic confirmation 

## Design Goals

- Deployability: Tor must not be expensive to run, must not place heavy liability burden on operators, and must not be difficult or expensive to implement
- Usability: Tor should require as few configuration decisions as possible, and support all common platform
- Flexibility: Protocol must be flexible and well-specified
- Simplicity: The protocol's design and security parameter must be well-understood

## Threat Model

Assume an adversary who can observe some fraction of network traffic; who can generate, modify, delete, or delay traffic; who can operate onion routers of his own; and who can compromise some fraction of the onion routers.

**Note:** Not global passive adversary!

## Tor Design

Tor network is an **overlay network**, with each **onion router (OR)** running as a normal user-level process without any special privileges. Each OR maintains a TLS connection to every other onion router (every ??). Each user runs local software called an **onion proxy (OP)** to fetch directories, establish circuits across the network, and handle connections from user applications. 

### Keys

Each onion router maintains a long-term identity key and a short-term onion key. Identity key is used to sign TLS certifications, OR' *router descriptor*, and if it is a directory server, to sign directories. The onion key is used to decrypt request from users to set up a circuit and negotiate ephemeral keys. TLS protocol also by default establishes a short-term link key when communication between ORs. Short-term keys are rotated periodically and independently to limit the impact of key compromise.

### Cells

Traffic passes along the connections in fixed-size cells, which is 512 bytes, consisting of header (with circuit identified, and command that describe what to do with the cell's payload), and the actual payload. Relay cells also have an additional header with streamID, end-to-end checksum for integrity checking, length of relay payload, and relay command. The entire contents of the relay header and relay cell payloads are encrypted or decrypted together as the relay cell moves along the circuit using 128-bit AES cipher in counter mode.

### Circuits 

In Tor, each circuit can be shared by many TCP streams. To avoid delays, users construct circuits preemptively. A user's OP constructs circuit incrementally, negotiating a symmetric key with each OR on the circuit, one hop at a time. To begin creating a new circuit, Alice's OP sends a *create* cell to the first node in their chosen path. say Bob. Following which, the Alice sends a *relay extend* cell, of which Bob will pass it to Carol to extend the circuit. When Carol responses with *created* cell, Bob wraps the payload into a *relay extended* cell and passes it back to Alice. Now the circuit is extended to Carol, and Alice and Carol (like Alice and Bob) share a common key. This behavior continues until the full path is established.

This circuit-level handshake protocol achieves unilateral entity authentication (Alice knows she’s handshaking with the OR, but the OR doesn’t care who is opening the circuit— Alice uses no public key and remains anonymous) and and unilateral key authentication (Alice and the OR agree on a key, and Alice knows only the OR learns it).

Once Alice has established the circuit (so she shares keys with each OR on the circuit), she can send relay cells. Upon receiving a relay cell, an OR looks up the corresponding circuit, and decrypts the relay header and payload with the session key for that circuit.  If the cell has a valid digest , it accepts the relay cell and processes it.

To tear down a circuit, Alice sends a destroy control cell. Each OR in the circuit receives the destroy cell, closes all streams on that circuit, and passes a new destroy cell forward. But just as circuits are built incrementally, they can also be torn down incrementally


## Top Changes in Tor Since Original Paper (2004)

Link: <https://blog.torproject.org/top-changes-tor-2004-design-paper-part-1/>

1. Node discovery and directory protocol

Users need to be able to find and select nodes with the same probability distribution, otherwise an adversary will be able to exploit differences in client knowledge. 

We can't depend on just asking our peer for list of known relays, because if there's a malicious relay then it will only tell you relays that it controls.

The original model uses "directory" object, which is a signed **router descriptor**, that made available to a small set of known *directory authorities* and is periodically queried and downloaded by users. This system had severable notable problems, including:

- Clients needed to download the same descriptors over and over, whether they needed or not
- Each directory authority was trusted outright, and so any one misbehaving could completely compromise all the clients that talked to it.
- The load on authorities will likely grow to be excessive

**Changes for this include:**

1. Improving scalability: Directory servers also provide a list of which authority servers are down. 
2. Caching: Nodes are allowed to act as directory cache, and we ensure integrity by having authorities sign their results
3. Trust model: To get router descriptors, clients would contact one or more caches, and ask for the descriptions they believed represent the consensus view of all of the authorities
4. Consensus: Authorities would exchange vote documents between themselves periodically, compute a consensus document, and have all of them sign the consensus. This means client would only need to download a single signed consensus document periodically.

2. Security improvement for hidden services

We want to be able to support hidden services. (To be honest I don't understand what this part is about) There's something about introducing **Introduction Points (IP)** where client and server can meet with each other, and then they will communicate over a shared **Rendezvous Point (RP)**.

4. Cell queuing and scheduling

The original Tor design left the fine-grained handling of incoming cells unspecified: every circuit's cells were to be decrypted and delivered in order, but nodes were free to choose which circuits to handle in any order they pleased.

**Changes:** Instead, Tor currently places incoming cells on a per-circuit queue associated with each circuit. Rather than filling all output buffers to capacity, Tor instead fills them up with cells on a near just-in-time basis. Also, they also started favoring circuits on each connection that has been quit recently (so circuits with small, infrequent amount of cells will get better latency than circuit being used for bulk transfer)

5. Guard Nodes

They now assume, based on previous research, that if an attacker controls or monitors the first and last hop of a circuit, then the attacker can de-anonymize the user by correlating timing and volume information. 

To improve this situation, the guard node feature was implemented. What this do is that the Tor client picks a few Tor nodes as its "guards", and uses one of them as the first hop for all circuits (as long as these nodes remain operational). 

**Why does this help?** While this choice doesn't affect the probability that the first circuit is compromised, it does mean that if the guard nodes chosen by a user are not attacker-controlled all their future circuits will be safe. On the other hand, users who choose attacker-controlled guards will have about `M/N` of their circuits compromised, where M is the amount of attacker-controlled network resource and N is the total network resource. Without guard nodes every circuit has a `(M/N)*2` probability of being compromised.

Essentially, the guard node approach recognizes that some circuits are going to be compromised, but it's better to increase your probability of having **no** compromised circuits at the expense of also increasing the proportion of your circuits that will be compromised if any of them are. 

6. Censorship Resistance 
 
While Tor was originally designed as an anonymous communication, over time it was also often use to circumvent censorship and censors from blocking access to certain websites.

While the anonymity offered by Tor helps with this issue, this isn't enough. It was easy for censors to block access to the whole of the Tor network in the original design (because there were a handful of directory authorities that user need to connect before they could discover addresses of Tor nodes). Censors can even go as far as blocking IP address of all Tor nodes.

**Solutions:**

One way this has been dealt with is introducing **bridges**, which are special Tor nodes that are published in the directory, and could be used as entry points to the network. Users needed a way to find about these somehow, so the bridge authority collect bridge IP addresses and distribute to users through alternative platforms (like email, on web, via personal contacts, etc.)

In addition, Tor has gradually changed its TLS handshake to better imitate web browser traffic (as to avoid censors that use traffic analysis). Currently, more work is being done to transform Tor traffic using pluggable transport design that obfuscates the traffic.