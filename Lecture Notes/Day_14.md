# SSL (Secure Socket Layer)

## Reading Question

Why was there support for SSL 2.0 within SSL 3.0? Because at the time SSL 3.0 came out everyone was using SSL 2.0, so it would too world-breaking if SSL 3.0 is incompatible with SSL 2.0, however ensuring compatibility and security is difficult (even certain versions of TLS have issues with this)

We know that TCP/IP doesn't provide authenticity and confidentiality guarantees. Thus it's necessary for us to use crypto create a secure channel on top of TCP/IP. 

SSL builds on the concept of the **secure channel**. A secure channel is intended to allow a connection between two clients on the Internet and adversary in the middle can't be able to peer into the actual data being transmitted within the channel.

Running unsafe protocols like HTTP, SMTP, IMAP/IPOP help make them "secure". Over time SSL has progressed to be called TLS (its improved version). SSL 3.0 isn't used anymore due to certain security flaws we've since found in it.

## Cryptographic Primitives

We'll discuss the general workings of symmetric and asymmetric key encryption.

With asymmetric key encryption, we can:
- KeyGen() -> public key `PK`, secret key `SK`.
- Encrypt(`PK`,message `m`) -> ciphertext `c`
- Decrypt(`SK`,`c`) -> `m`
  - or we can do: 
  - Sign(`SK`,`m`) -> signature `sig`
  - Verify(`PK`,`m`,`sig`) -> okay?
Examples include: AES, DES

With symmetric key encryption, we can:
- KeyGen() -> private key `K`
- Encrypt(`K`,`m`) -> `c`
- Decrypt(`K`,`c`) -> m
  - or we can do:
  - MAC(`K`,`m`) -> tag `t`
  - MAC is used for authenticating the identity of a user, and also to verify that no one has changed the message. Assuming both sides of the connection have the same private key `K`, by sending the tag `t` over with the encrypted message `c`, the receiver can first decrypt the message, recompute the tag with MAC(K,m), and verify it is the same as what was transmitted over
    - This makes it so that an adversary won't be able to get the client to accept a malicious message without a proper tag, which they don't have the private key `K` to computer.
  - The lecture notes says that: Keys for MAC and encryption should be different. If we need both encryption and MAC, we will need to generate two keys.
Examples includes: RSA, EC, Diffie-Hellman (DH)

Usually we use asymmetric key operations to send over a symmetric key which is then primarily used for actual data encryption. This is because symmetric key operations are often much faster than asymmetric key operations.

## Strawman protocol

Goal: establish a shared key for symmetric crypto
Process:
1. C --> S: connect / establish a connection
2. C <-- S: my public key is PK_S
3. C --> S: Encrypt(PK_S, freshKey)
4. C <-> S: Encrypt(freshKey, m...)
   1. Both sides use symmetric key to send encrypted messages.

What are problems with this protocol?
- **MITM Attack:** Adversary intercept messages to the client and embeds with its own symmetric private key (step 2). The adversary can then intercept the encrypted messages from the client, decrypt them, and then send them to the server using its own connection. The data flow back also works the same way.
  - In the perspective of the client and server everything seems to work fine. 
  - **Solution:** Use certificates. Everyone maintains a list of trusted authority server, which maintains a table of (principal name, public key) pairs. The principal name is typically the server's DNS name. The client will then talk to the certificate authority to get the public key for the server.
  - However, in practice this method is not very performant, since we're putting the CA on every critical network path.
    - Instead, the authority server *pre-signs* a message:
      - **Sign(SK_CA, {server, PK_server})**
      - This signature is then used as a server's "certificate", which it can then send to to the client at the start of the connection.
      - The client, which has built-in knowledge of the CA public key, can verify that the message correctly signed by the PK_CA.
    - Revised strawman protocol:
      1. C --> S: connect
      2. C <-- S: PK_S, Sign(SK_CA, {server: PK_S})
      3. C --> S: Encrypt(PK_S, freshKey)
      4. C <-> S: Encrypt(freshKey, m...)
- Decryption of previous messages upon exploitation of key: If the private key is somehow leaked sometime in the future, an adversary which has been keeping track of packets prior will be able to figure out the packet's content.
  - **Solution:** Establish **forward secrecy** property, which assures that session keys transmitted over key agreement protocol will not be compromised even if the key uses used for the key exchange are compromised.
  - Instead of using long-term keys for encryption, we use short-lived keys for encryption, and long-term keys for signing. Thus we'll generate a new public-private key pair at the start of every connection (ephemeral keys)
  - Strawman forward secrecy for our protocol:
  1. C --> S: connect
  2. C <-- S: PK_conn, Sign(SK_CA, {server: PK_S}), Sign(SK_S, PK_conn)
     1. Notice the use of long-term key SK_S for signing message that client can use to verify the trustability of the ephemeral key PK_conn
     2. So we use SK_CA to trust PK_S, and then we use PK_S/SK_S to make us trust PK_conn
  3. C --> S: Encrypt(PK_conn, freshKey)
  4. C <-> S: Encrypt(freshKey, m...)
  - Thus as long as the server properly deletes the ephemeral key after the connection closes, even if the long-term secret keys are exposed there will be no way of recovering `freshKey`. 
      
## SSL (Secure Socket Layer)

SSL has issues with versioning / compatibility, cipher integration (properly negotiating which cryptographic method to use), working with servers having many certificates.

SSL 3.0 handshake (simplified model):

  1. C -> S: Hello: client version, randomC, session_id, ciphers
     1. `ciphers` is list of ciphers that the client is willing to use
  2. S -> C: Hello: server version, randomS, session_id, ciphers
     1. **Note:** Both messages in step 1. and 2. is that there is *no* authentication of these messages.
  3. S -> C: ServerCertificate: cert list
     1. For some servers would be a list of certificates the server maintains
  4. S -> C: HelloDone
  [[ client picks random `pre_master_secret` ]]
  5. C -> S: ClientKeyExchange: encrypt (`pre_master_secret`, PK_S)
  6. C -> S: ChangeCipherSpec
     1. Denote to change over the agreed cipher.
  7. C -> S: finished, MAC({master_secret ++ msg 1,2,3,4,5}, C_mac_key) (and encrypted)
     1. `MAC({master_secret ++ msg 1,2,3,4,5}` - used by client to say "this is everything I believe you have told me so far" in the form of a MAC. The server then needs to verify this information and make sure what it got matches what the client got.
     2. Adversary can't modify this properly because it doesn't know the `pre_master_secret`. Also note it may be possible to try to attack the MAC algorithm but security is still provided as long as the MAC can't be attacked in real-time (since connections usually time out on the order of a few minutes)
     3. This technique prevents **downgrade attacks** on the SSL protocol.
  8. S -> C: change cipher spec
  9.  S -> C: finished, MAC({master_secret ++ msg 1,2,3,4,5,7}, S_mac_key) (and encrypted)
      1.  Same "trick" as in step 7.
  10. C -> S: encrypt({data, MAC(data, C_mac_key)}, C_enc_key)
  master_secret <- PRF(pre_master_secret, "master secret", randomC+randomS)
  (C_mac_key, C_enc_key) <- PRF(master_secret, "key expansion", randomS+randomC)

  
SSL 2.0 attack
- Adversary could edit ClientHello message undetected
  - Tell S to substitute strong ciphers with weak ciphers

SSL 3.0 risk: version roll-back attack
- Adversary modifies ClientHello to be say version 2
- Clients adds "marker" in  authenticated padding bytes
- Works only with RSA key-exchange

SSL 3.0 attack: drop ChangeCipher
- MAC doesn't include message 6 and 8 (for some compatibility reason)
- Thus allows for the attacker to delete message 6 and 8. The effect is that Client and server won't switch to using ciphers!
- Attacker can change data to S
- Security modification made in**TLS 1.0:** Finish must be preceded by ChangeCipher


## Caveats of Secure Channel

- Data confidentiality: Data is encrypted, and so this helps ensure that the data transmitted over the connection is confidential to only the two participating parties. 
  - However, even with SSL or TLS we can still gauge the approximate size of the data being sent (which the adversary can use to guess what kind of information you're sharing). That said, you can also add padding to somewhat limit the amount of information that can be learned.
  - The adversary can also learn information about the timing of when messages are sent.
- Integrity: Things are pretty good, but TLS doesn't really authenticate the EOF. It may be possible for the attacker to force a connection to be ended prematurely (as we saw by guessing the sequence number in TCP).
- Privacy: Even with TLS information about the IP address (who you are talking to) and service name is still leaked.
