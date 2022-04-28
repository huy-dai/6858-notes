## Secure Messaging

Link: <https://css.csail.mit.edu/6.858/2022/readings/secure-messaging.pdf>

## Threat Model

- Local adversary: An attacker controller local network
- Global adversary: An attacker controlling large segments of the Internet, such as powerful nation states or large ISPs
- Service providers: Potentially the company controlling the centralized infrastructure of the messaging system

## Desirable Properties

1. Security and privacy
2. Usability
3. Adoption considerations

## Problem Areas

The paper defines three orthogonal problem areas for secure messaging: 
1. Trust establishment problem: Ensuring the distribution of long-term keys and proof of association with the owning entity,
2. Conversation security problem: Ensuring the protection of exchanged messages during conversations
3. Transport privacy problem: Hiding the communication metadata 

## Trust Establishment

Trust establishment is where the users verifying that they are actually communicating with the parties that they intended to. **Key exchange** is the process where users send cryptographic keys to each other, while **Key validation / key verification** is the mechanism allowing users to ensure that cryptographic long-term keys are associated with the correct real-world entities. These two process combined makes up trust establishment.

Trust establishment help fight against MITM attacks, allow for key revocation, and preserves user privacy.

Overall evaluation is that usability and security often comes at-odds with each other. Authority-based trust is predominantly used among recently developed apps in the wild, as well as among apps with largest userbases.

Transparency logs might provide more accountability with no interaction from most users. However, because this approach has not yet been deployed, it remains to be seen how much security is gained in practice.

## Conversation Security

Conversation security protocol protects the security and privacy of all the exchanged messages. This encompasses how messages are encrypted, what data is attached to them, and what cryptographic protocols (e.g. ephemeral key exchanges) are performed.

Conversation security help ensured confidentiality and integrity of messages. In addition, users can be assured of authentication (knowing the party they are talking is still the same person), forward secrecy (compromise of key material does not enable decryption of previously encrypted data), backward secrecy (compromise of key material does not enable decryption of succeeding encrypted data), message repudiation (can't tell a given message was authored by any particular user).

Discussion: **Static asymmetric cryptography** (used for signing *and* encrypting) provided confidentiality, message authentication, and integrity, but it losses all form of repudiation, and there's also a lack of forward or backward secrecy. However, we can mitigate the later by having keys with very short lifetimes. There's also an interesting method called **key evolution**, in which the initial session key can evolved over time through the use of *session key ratchet*. This provides forward secrecy but not backwards secrecy.

Currently most non-security focused applications provides only the basic static asymmetric cryptography. Another outstanding concern that limits the adoption of secure conversation security protocols (outside of implementation difficulty), is the limited support for multiple devices. Nowadays users often own multiple devices, but device pairing process has shown to be extremely difficult for users in practice, and allowing users to register multiple devices with distinct keys is a major usability improvement.

## Transport Privacy

Transport privacy defines how messages are exchanged, with the goal of hiding message metadata such as the sender, receiver, and conversation to which the message belongs. Some transport privacy architectures impose topological structures on the conversation security layer, while other simply add privacy to data links between entities.

Transport privacy can ensure sender and/or recipient anonymity, participation anonymity, unlinkability (no two global entities except the participants can discover that two protocol messages belong to the same conversation), and resistance to global adversary.

Certain desirable adoption properties include being topology indepndent, being resistent to spam/flood, using low network bandwidth and computation, and is scalable.

Discussion: **Store-and-forward** is the baseline technique usedby email and text messaging, where you send message to intermediate server with all sender and recipient information, and that message is later forwarded on. This does not guarantee any privacy properties. **Onion routing** is a method for communicating through multiple proxy servers that complicates end-to-end message tracing. However, onion routing anonymity properties can still be broken by global network adversaries. Onion routing is also vulnerable to DOS attacks. 

If messages are secured end-to-end, leaving only identifies for anonymous inboxes in the unencrypted headers, then metadata is easily hidden from service operations. However, this scheme are prone to spam, flooding, and DOS attacks, and require expensive operation such as zero-knowledge authentication, which pose barriers to adoptions.