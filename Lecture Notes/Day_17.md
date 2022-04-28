# Messaging Security

In messaging, we want to guarantee properties like:
- Confidentiality
- Authenticity
- Privacy 
- Deniability (Repudiation)

## How does email compare?

An email flow might look like the following:

Alice -> Email client -> Gmail -> (over SMTP) mailing list -> (over SMTP) bob@ibm.com -> (forwarded over SMTP to) MIT -> CSAIL + MIT's archive server -> Spam filter -> CSAIL again -> (over IMAP or POP4) Email client -> Bob

By and large, security in emails hasn't been as strong as other communication platforms. This is because security properties has been added on to a base layer which weren't designed for security (need for retrofit + backwards compatibility). Thus many of the security features added on to email has been through incremental changes, like introducing **per-hop TLS**. Additionally, there's a lack of strong guarantee of security, since email servers will automatically downgrade its security features if an email is failed to sent.

You can see how this is very similar to the situation we see with security on the Internet as a whole. TLS was added as a separate layer on top of the original TCP protocol, which hasn't seen much change since its early days except for some small changes (like in how it generates session values for security). 

Most of the security mechanisms we see today deals with **spam / phishing attacks**. Nowadays services like Gmail use protocols like DKIM, which have the sending servers sign the message to verify that the message did indeed come from the specific domain server.

## End-to-End Encryption Scheme

Cryptographic primitives
- m = plain text message
- h = Hash(m)
- tag = MAC(key, m) -- symmetric, essentially Hash(key||m)
- c = E(key, m) -- symmetric encryption
- m = D(key, c) -- symmetric decryption
- c = E(PK_Bob, m) -- Alice encrypts with Bob's public key
- m = D(SK_Bob, c) -- Bob decrypts with his private key
- sig = Sign(SK_Alice, m) -- Alice signs with her private key
- ok = Verify(PK_Alice, m, sig) -- Bob checks signature with Alice's public key


* PGP-like scheme
  * Assume both users know each other's public keys (PK_Bob, PK_Alice). 
  * Each user use the other's public key to encrypt their message, append to it their signature signed with the ciphertext, and sends it over to their target. 
  * The recipient verifies the signature and decrypts the message.
  * **Difficulty:** Alice and Bob need to get each other's public key (done through trust establishment protocol).
    * Another tricky thing is that you **shouldn't sign with ciphertext**, but rather with the original message. That way a MITM adversary can't replicate the message with their own signature and replay the original ciphertext.
    * Also, the signature should also specify the intended recipient! That way we make sure that the message sent / the ciphertext can't be replayed in some other context.
* Secure Channel (Simplified TLS)
  * Use asymmetric encryption to to share a secret symmetric key and use that for encryption in the rest of the communication. 
  * However this still encounters issue with trust establishment. Also it doesn't guarantee forward secrecy

## Trust Establishment

- Opportunistic encryption (SMTP TLS):
- E.g. secure channel starts with
    - A -> B: PK_A
    - A <- B: PK_B
    - ...
  - Each party sends its public key over the network to the other
  - This is a convenient option and fights passive attackers, but vulnerable to MITM attack (injection with attacker's key)
- **Trust on First Use (TOFU)**
  - Assume first connection was OK, and remember  pin each other's public key.
  - This is not useful against long-term active attacker, and there's no easy way to deal with when there's a key change (how do we know it was actually intended or a malicious change?)
- Out-of-band exchange:
  - We exchange public keys in person in advance, perhaps via QR code, or verify, afterwards, that TOFU pinned key is correct in person
  - But this is not easy to do because you need to meet in person - very cumbersome process
  - It's also not easy to deal with key changes.
- **Server assisted** (directory)
  - Users will need to authenticate to a central server, which then will allow users to query the central server for the other users' public key 
  - This increase usability for the users since the central server handles the key management and authentication. 
  - However, a compromised server will be very bad since it may lead to maliciously sending bad keys to server. That said, since the user still keeps their private key, the central server won't be able to decrypt past users messages.
- **Keybase**
  - A cool startup for allowing to use public keys across different websites
  - Server holds, for each name like "Alice":
    - `PK_Alice` (only Alice's devices have private key)
    - Identity records linking Alice to her account names on other services
      e.g. `Sign(PK_Alice, "I am alice177 on twitter")`
  - For each identity record,Alice must put signed statement visible in that other account. E.g. tweet it (encoded in ascii), post it on github, &c.
- Key Transparency
  - For each key server, maintain a log that tracks all key changes. Used to detect a corrupt key server or unauthorized key change. 
  - Key owners periodically check the logs to ensure that their key is correct in the log. The key requesters can also check that their result matches latest update in the log (perhaps periodically to limit traffic).
  - **Problem with performance** in terms of downloading log or looking through log efficiently.
    - Also note that key transparency reveals dishonesty, but doesn't enforce it (requires work from the user and server side to detect).

## Conversation Security

We want to guarantee **authenticity**, but at the same time you also want to provide some form of **deniability** (there are tradeoffs to be made between these two). Currently there's no agreement on what "secure messaging" actually means. 

In addition, we want to have high **confidentiality** but also allowing to deal with **spam/harassment**. 

### Deniability

We want to allow messaging such that user B can verify a message to be coming from user A but no one else can prove that A indeed sent that message.

**Idea:** Alice does not sign anything. Instead, use a MAC encryption -- MAC is symmetric, so anything Alice MAC'd, Bob could also have MAC'd. If we had signed with our private key then it directly points to the fact that the person sending the message does in fact have access to the full public-private key.

A simple deniable authenticity protocol, for Alice to send M to Bob could be:

1. Bob chooses random key `K` (e.g. K <- KeyGen()).
2. A <- B: E(PK_Alice, K)
3. Alice decrypts K.
4. A -> B: M, `MAC(K, M)`, where M is some encrypted message that B can decrypt.
5. Alice publishes `K`.
   1. Intentionally reveals the secret so that anyone in the public could use it to falsify the MAC signatures to be coming from Alice.
6. Afterwards, anyone could produce these message, so no proof it was Alice.
   1.  Everyone now knows K.
   2.  Anyone can produce E(PK_Alice, K).
       1.  This it's possible for anyone to produce a message `m'` and use PK_Alice along with K to say that Alice was the one who sent this message (spoofing other header parameters).
       2.  Similar since there's no way to prove that any released key is correct, another client can generate a different key `K'` and announce that Alice is the one who published it & send messages using K as Alice.
       3.  However, in the original message Bob can assured that the message came from Alice by checking it can replicate the MAC using the `K` it provided to Alice.

### Forward Secrecy

Basic plan:
- Bob wants to send Alice message M with forward secrecy.
 - Alice generates temporary public/private key pair PK_temp, SK_temp.
 - A -> B: PK_temp
 - A <- C: E(PK_temp, M)
 - Alice decrypts C with SK_temp.
 - Alice discards SK_temp (overwrites all copies with junk).
 
This scheme is usually combined with a secure channel scheme like TLS to provide properties such as authenticity, using the long-term public keys PK_Alice and PK_Bob. As an extension, we can even another layer by using the temp pub-private key pair to send over an ephemeral symmetric key to use for actual encryption (since more performant).
  
* Why can't the attacker later decrypt recorded E(PK_temp,M)?
  * This is because the attacker needs SK_temp, which is no longer maintained on Alice or Bob's computer, so useless for attacker to break in.

## Transport Privacy

We want to protect metadata (e.g. who is the sender, who is the recipient). This is a costly thing to implement, and in fact most modern systems don't bother with this aspect. 

* Instead of sending message directly from A to B, send to the central server which will do the routing instead. However, this scheme is vulnerable to a compromised server. In addition, an attacker may still be able to guess who is talking to whom through timing of msgs (I see a message coming in from A, and then a message coming out to B soon after). Can derive better guesses from probability + analyzing message traffic over time.
* Onion routing: Instead of one server, we relay messages through many servers. This makes it much more difficult to follow packets, but a **global adversary** may still be able to trace messaging timing (e.g. at the node's ends).
* Broadcast: Send message to everyone, and only the intended recipient will be able to decrypt it. This is good privacy but very bad performance (we do have mixnets which process large batches of messages at the same time).