# Security Problems in TCP/IP Protocol Suite

Link: <https://css.csail.mit.edu/6.858/2022/readings/lookback-tcpip.pdf>

Author is Steven Bellovin who worked in Bell Labs in the 1980s, and was also a early adopter/ researcher of the early Internet.

The TCP/IP protocol suite was developed under the Department of Defense. When describing attacks of TCP/IP, they assume the attacker has complete control over some machine connected to the Internet. Also, in the paper they will be discussing generic issues with TCP/IP and not implementation or vendor-specific protocols.

## TCP Sequence Number Prediction

Usually in a 3-way handshake to establish TCP connection, the following happens:
1. Client selects and transmit an initial sequence number `ISN_C`
2. The server acknowledges it and sends its sequence number `ISN_S`
3. Client acknowledges that number, and then proceed to send data.

However, suppose an adversary X does step 1 impersonating as the client. In step 2 the sever would then send the respond with `ISN_S` back to the intended client. If the adversary is able to **predict** `ISN_S`, then in step 3 the adversary can acknowledge the server `ISN_S` as the client and then be able to send data to the server thereafter.

How does one predict the random ISN? In Berkeley systems, the initial sequence number variable is incremented by a constant amount once per second, and by half that amount each time a connection a connection is initiated. Thus, if one is able to initiate a legitimate connection and observes the `ISN_S` uses, one can calculate with a high degree of confidence what `ISN_S` will be at the next connection attempt. Another way to get this `ISN_S` comes from the `netstat` protocol, which exposes a lot of info about a server to the Internet.

Also, note in step 2 the server sends a message to the impersonated client. Usually this would cause the real T to receive it and attempt to reset the connection. However, Morris found that he was able to get around this by overflowing queues on the way such that the message would be lost. 

An important point, according to the author, is that **address-based authentication is very vulnerable to attack. TCP was never designed to be a secure protocol**. In this case about the sequence numbers, there were any guarantees at the start about the properties that they were supposed to have, and so different interpretations led to security holes.

### Resetting connections

In 2004, Watson found that TCP reset packets would be used if the RST bit was set on a packet whose initial sequence number was anywhere within the receive window. On modern systems, the receive window is often 32K bytes or more, so it would take less than `2^17` trials to generate such a packet in a blind attack, but being to reset any long-lived sessions (like BGP sessions between routers) can have wide-spread effects on the global Internet routing table.

### Defenses

To fight against this attack, the better way is to randomize the increment. A combination of a fine-granularity increment and a small random number generator is good. However, an even better way is to use a cryptographic algorithm for `ISN_S` generation. Another defense includes having good logging and alerting mechanisms.

At the end of the day though, the best thing is that **don't rely on TCP sequence numbers for security**.

## Routing Security Problems

The easiest mechanism to abuse in routing is IP source routing. This can occur when the target uses uses the reverse of the source route provided by the attacker in a TCP open request. Through this, the attacker can then pick any IP source address desired, including that of a trusted machine on the target's local network, and any facilities available to such machines become available to the attacker.

### Defenses

One way to defend is to have gateways reject external packets claiming to be from local network. However, this can impact functionality of some higher-level protocols or networks where you have multi-organization backbone that peer to each other. The better solution is to use **firewalls**. Note: **Don't trust firwall fully, since they don't defend against insider attacks**

## Routing Information Protocol (RIP) attacks

RIP is used to propagate routing info on local networks. Typically, the information received in unchecked, which allows an introducer to send bogus routing info to a target host, and to each of the gateways along the way, in order to impersonate a particular host. This allows for all traffic destined for real user to be sent to the introducer's machine. An additional layer can be added to trick the real user to also send to the intruder, so they see traffic both ways. 

This attack remains a very serious threat. In the "AS 7007" incident, an ISP started advertising that it has the best routes to most of the Internet. This caused a disruption in the global routing tables that took over 4 hours to stabilize.

### Defenses

A gateway that filters packets based on source or destination address, will block any form of host-spoofing, since the offending packets can never make it through. Another defense is for RIP to be more skeptical about the routes it accepts. In addition, it would be useful to **authenticate RIP packets**. At the time of paper, they say pub key signature schemes are difficult to implement. However, even if this is done, the receiver can only authenticate the immediate sender, which in term may have been deceived by gateways further upstream.

## Exterior Gateway Protocol (EGP)

EGP is intended for communications between core gateways and exterior gateway. The exterior gateway is periodically polled by the core gateways for info about the network it serves. Also, the exterior requests routing info from the core gateway. While it's difficult to inject a false route update since there the amount of responses that is allowed to be received is limited, one attack is to impersonate a second exterior gateway for the same AS (though potentially defended with trusted list of gateways). Another way is to have the attack claim to be a gateway that claims reachability for some network while the real gateway intended to serve it is down. This would allow password captures by assorted mechanisms.

### Defenses

Set topological requirements that exterior gateways must be on the same network as the core, so that intruder will need to have control over machine on the local network. 

**Note:** Due its restrictions, EGP was secure but not performant, so it has been replaced with BGP.

## Internet Control Message Protocol (ICMP)

The ICMP Redirect message, which is used by gateways to advise hosts of better routes, can be abused like RIP to replace good routes with bad ones. There's some more limitations here though in terms of where the redirects can be sent from. Many hosts however also do not perform enough validity checks on ICMP Redirect Messages.

ICMP messages with error flags can be used to reset existing connections. An even more global DOS attack can be launched with a fraudulent Subnet Mask Reply message, which can cause devices to block all communications with the target host.

### Defenses

Add checks to ICMP packets regarding sequence numbers, along with restricting route changes for these traffic, or even considering not using it at all.

**Note:** The easier and better types of attackers are with ARP-spoofing rather than ICMP attacks.

## Authentication Server

This is the model of the client having an authentication server which web servers can contact to verify identify of the user. This is used over address-based authentication. A lot of the described risks and attacks are dealt with the authentication server not being trusted or it failing or being compromised by routing table attacks.

### Defenses

Servers should use better methods for authentication. TCP itself is inadequate.

## Susceptible protocols

- `Finger` service are services that display useful information about users. They can be used to exfiltrate info
- Mail servers at the current time doesn't provide any authentication mechanism, which leaves the door wide to faked messages. However, even so, we're working with the policy that anyone, including spammers, can send us email messages.
- Normal passwords are too easy to be stolen. A better option is to use one-time passwords (pub-key authentication)
- DNS. Attacks can cause bad DNS translation to malicious websites or send all traffic to attacker for interception. Even things like a **zone transfer (AXFR)** request can be used to download an entire section of the DNS database (this leaks info). The author supports the use of Kerberos system for authentication of queries and responses.
- FTP sends password in cleartext. Anonymous FTP is a huge security problem.
- SNMP authentication, at least early versions, send cleartext passwords.
- DHCP is better over older protocols, but DHCP authentication is still rarely used.
- ARP attacks are overall easy to launch and hard to spot. ARP have been used to send broadcast storms.


## Comprehensive Defenses

1. Authentication: Don't just trust IP source address for authentication. Use cryptographic authentication. Authenticate over DNS to provide more trust, for example.
2. Encryption: At time of writing, encryption devices are expensive, often slow, and hard to administer. There's also different levels of encryption, from link-level encryption, and application layer encryption, and end-to-end encryption. However, these have issues with key distribution and management.
   1. Also using better checksums than just TCP checksums (which is rather weak) to avoid attacker substituting data with malicious code with same TCP checksum\
3. Use trusted systems model. Establish different levels of trust within your network and also outside of your network. Use MAC or DAC