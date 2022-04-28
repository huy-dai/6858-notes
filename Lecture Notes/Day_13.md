# Network Security

Today's lecture is focused on attacks on core network protocols. In later lectures we will be discussing host to host cryptographic solutions.

## Current Internet Model

In the current model of the Internet we're working with an open network -e.g. anyone can join- consisting of many ISPs that is heavily decentralized containing many protocols (TCP, BGP > RIP, DHCP > ARP, UDP, Telnet, FTP, IMAP > POP, etc). A difficult aspect of this model is that it is very complex to implement and enforce security across the Internet.

The Internet today has made much progress in providing security at the higher levels, from Kerberos to SSH to SSL. However, the core network components doesn't provide security, and these components can't necessarily be changed due to the need for systems to be backwards compatible. Nowadays a lot of the security features are implemented at the host-to-host / application level. However, we won't worry about those parts for today, and instead focus on fixes made with the core network.

Example layer:
1. App 
2. TLS
3. TCP, UDP
4. IP

We'll focus on part 3 and 4 for this lecture. Note that we'll also discuss protocols, which, like core network, is **very slow to evolve.**

## TCP Handshake

1. In the first step, the client sends a packet with fields:
   1. Source: C
   2. Dest: S
   3. SYN(SN_C)
2. Server then receives the packet and sends back:
   1. Source: S,
   2. Dest: C,
   3. SYN(SN_S), ACK(SN_C)
3. Client then acknowledges server sequence number:
   1. Source: C, 
   2. Dest: S, 
   3. ACK(SN_S), SN_C 
      1. **Note SN_C is also included from the client back to server**
4. Then finally the client sends actual data:
   1. Source: C
   2. Dest: S,
   3. Data(SN_C+1), ACK(SN_S)

There's no cryptography or signature involved here. The security provided here is that the server will maintain a table associating client sequence number (SN_C) with the provided server sequence number (SN_S). Thus for the server to take in the client's data it will require that client will need send acknowledgement of SN_S. In essence the sequence number provides a form of challenge-based authentication. 

Also there are also a number of logic control behind the scene that will automatically resend packets - for example, if the server ACK packet of SN_C in step 2 is dropped in the network layer, the server will resend the packet after some timeout.

### TCP Spoofing

As mentioned in the paper, one way to do TCP spoofing is to send a packet that initiates a connection to the server impersonating as the client. For this to work, the only thing we have to know is to predict / know with a high probability the server sequence number `SN_C`.

How bad is this? In theory this allows the attacker to impersonate a user and establish connections as them. The most worrisome services are with those that uses IP-level authentication. Some protocols mentioned in the paper like `rlogin` is outdated, but you can also use this for modern services like SMTP to send spam as a user or use it to get MIT campus access. In addition we can also execute DoS attacks by resetting connections or injecting malicious data into connection.

### Choosing SN_S

```c
uint32 isn;
every second:        isn+= 128;
upon new connection: use sn_s= isn;
                     isn += 64;
```
How do you get SN_S?
1. First, connect to the server to get `SN_S1`.
2. Spoof and guess: `SN_S = SN_S1 + 64`.

A more sophisticated method is to do statically analysis on the change of SN_S over time and create a model of the behavior of the traffic to better predict SN_S.

### Defenses
1. Use TLS for secure connections
2. ISP-level filtering
   1. Drop packets which ISP detects is mismatch from the source.
   2. However, this is only applicable
3. Establishing a per-host ISN. Nowadays the rules for choosing `SN_S` has been changed to be `SN_S = isn + H(src, dest, secret)` where H is a hashing function and secret is some extra text that only the server knows / generates.
   1. This way we still get the SN_S to increase linearly which makes it easier for us to detect out-of-date connections while making it difficult for attackers to guess SN_S.

Is it better not to trust sequence numbers at all and rely higher-level protocols like TLS for security? The issue is that if we don't attempt to fix the issue with guessing sequence numbers **it makes it trivial for attackers to reset TCP connections**. TLS only ensure your data is encrypted, but if we don't trust TCP at all then it will make it very difficult for us to ensure the *liveness* of the Internet if TCP spoofing can be done at any time to DOS users.

## DNS Spoofing

1. Client C sends to server:
   1. SRC = C,
   2. DST = S,
   3. data = mit.edu
2. Server responds back to client:
   1. SRC = S
   2. DST = C,
   3. data = 18.70...

Note that DNS is transmitted over UDP to decrease load on network. A big concern in this setup is the potential for an adversary to spoof the DNS response from the server. For example, if the adversary is able to detect or guess that the client is making a DNS query to some website it can send to the user a falsified DNS response pointing to its web page instead.

### Defenses

Most DNS implementations nowadays worry about this attack. One way they try to mitigate this is find someplace in the DNS packet to inject nonces as to make it difficult for attackers to spoof DNS response. This includes:
- SRC port: 16 bits
- Query ID: 16-bits

Deploy DNS SEC. Nowadays DNS SEC sort of fell out of favor as other technologies like TLS became available and addressed the similar set of issues that DNS SEC does, and the issue with DNS SEC is that there is an insignificant level of overhead that it adds to DNS traffic.

## DOS Attacks 

### SYN-Flooding

A form of DOS attack is SYN Flooding attack, which attempts to overflow the table which associates a client-provided sequence number with a server-generated sequence number. What an adversary can do is to send a large amount of packets corresponding to the first step of the TCP handshake and then never respond again. The server will attempt to store all of these entries in its table, eventually either i) running of memory and crash, or ii) decide to stop allocation after exceeding a certain threshold (usually 50) and block all new connections. 

It's really hard to defend against these attacks since the server won't know which connection is legitimate and which isn't, and IP-filtering doesn't quite work because the adversary can spoof to be different IP addresses.

### Defenses

One of the way to get around this is to change the requirement to not store state until a good connection has been established. In our case, we introduce **SYN cookies** that gets stored on the client. We set `SN_S = SN_C + [timestamp][H(src,dst,timestamp,secret)]`. In that case whenever we get a packet in step 1 of the handshake we can quickly calculate SN_S from SN_C and send it back to the client without storing any state. If we get a packet corresponding to the third step, which has to include ACK(SN_S) and SN_C, we can quickly take SN_S and SN_C, check the derived timestamp, and also perform a rehash to verify that the hash portion of SN_S is correct. This allows us to validate SN_S without storing state. Note the timestamp is very slowly incrementing, in the order of a minute.

### Smurf Attack - Bandwidth Amplification 

Over IP, there is a protocol named ICMP, which has an option to send an echo request (ping) packet to the *broadcast* address of a network. It used to be that you would get an ICMP echo reply from all machines on the network. One malicious way you can use this feature is to supply a victim's address instead (), which means the victim will get all replies instead of you. This allows that for a local subnet with 100 machines on a fast network, you will be getting a 100x amplification of each packet you send!

### Defenses

Nowadays, routers now block "directed broadcasts", meaning packets that are sent to broadcast address. 

### DNS Amplification

However, in modern day a new variant of Smurf attack has emerged called **DNS amplification**. This exploits the observation that a query to DNS is very small compared to the actual results you often get back. This gets even worse with DNSSEC, since their responses contains lots of signatures. Also, since DNS runs over UDP, the source address is completely unverified.

### Defenses

One way we can try to prevent this attack is to fix DNS servers to only respond to legitimate clients. However, this is hard to implement or design, since many name servers must necessarily respond to any open-ended set of clients (ex. laptops located off MIT campus, but configured with MIT DNS servers).

## Routing Protocol Attacks

There are two levels of routing protocols to worry about: 
- Those that apply to a local network and 
- Those that apply to the larger Internet. 
 
For local networks, protocols like ARP and DHCP have very little security, and they mostly uses broadcasts. For ARP, which is used to broadcast a request for a target's MAC, anyone can listen to such a broadcast and send a falsified reply (there's no authentication). Thus ARP allows adversaries to potentially impersonate router and intercept packets, even on a switched network. With DHCP, adversaries can also send clients false DHCP responses, which can affect a number of assignments from IP address, router address, DNS server, DNS domain list, etc.

For WAN networks, protocols like BGP applies. In BGP, any BGP participant router can announce route to any IP address. An attack that an adversary can perform is to announce that they have a route to a certain domain, and so people would then route through you (causing a BGP leak). This allows attackers to inspect/modify traffic before forwarding it to the destination domain. In the past BGP has been exploited / used to **send spam e-mails** to services like Gmail and Yahoo. The way this is done is that spammers will look for a block of IPv4 addresses not claimed on the Internet, proceed to advertise with BGP that they have the routing of that block of IP to their servers, proceed to send emails spoofing as those unclaimed IP addresses, and then move on when that group of IP has been blocked by spam lists.

### Defenses

Use better / more secure protocols including S-BGP, RPKI, BGPsec., MANRS. Techniques uses in these protocols include using sign original announcements, maintaining a trusted database of who is allowed to announce what IP prefixes, having signed paths, etc. 

## Wrap-Up

How to improve security in core?
- Protocol-compatible fixes to TCP implementations (and other protocols)
- Firewalls.
    Issue: adversary may be within firewalled network.
    Issue: hard to determine if packet is "malicious" or not.
    Issue: even for fields that are present (src/dst), hard to authenticate.
    TCP/IP's design not a good match for firewall-like filtering techniques.
    E.g., IP packet fragmentation: TCP ports in one packet, payload in another.
- Cryptographic security on top of TCP/IP: SSL/TLS, Kerberos, SSH, etc.
   - A hard problem: protocol design, key distribution, trust, etc.
   - Will talk about this more in next lecture on SSL.
- Some kinds of security hard to provide on top: DoS-resistance, routing.
   - Hard to implement since we're working with an open network model