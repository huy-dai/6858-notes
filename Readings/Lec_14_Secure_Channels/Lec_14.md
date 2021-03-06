# Analysis of SSL 3.0 Protocol

## Introduction

SSL protocol intended to provide a practical,widely applicable  application-layer + connection oriented mechanism for Internet client/server communications security. While SSL 2.90 has become the defacto for cryptographic protection of web HTTP traffic, it has several limitations which are addressed in SSL 3.0, which includes:

1. Weak MAC construction
2. Padding-length field is unauthenticated
3. Weakness to ciphersuite rollback attack (attacker forces both endpoints to use weaker form of authentication)

SSL is divided into two layers: 
- **SSL handshake protocol** (above): Is a key-exchange protocol that initializes and synchronizes cryptographic state at the two endpoints
- **SSL record layer** (below): Provides confidentiality, authenticity, and replay protection over a connection reliable transport protocol (like TCP)
  - Confidentiality: SSL protocols encrypts all application-layer data with a cipher and short-term session key negotiated by handshake protocol. Short-term sessions keys are generated by hashing random per-connection salts and a strong shared secret. Independent keys are used for each direction of connection as well as different instance of a connection.
    - The use of different keeps for different context and also using strong authentication on all encrypted packets help guard against **cut-and-paste attacks**.
  - Integrity: SSL cryptographically authenticates sensitive communications. SSL protects the integrity of application data by using a cryptographic MAC (usually HMAC), which is then signed with a long-term private key to avoid modification in transport.
  - Replay protection: SSL protects against replay attacks by including an implicit sequence number in the MACed data. Sequence numbers are 64 bits long, are maintained separately for each direction of each connection, and are refreshed upon each new key-exchange.
  - **Non-protections:** SSL does not protect against attackers deriving information from unprotected packet attributes like IP source & dest addresses, or examining volume of network traffic flow (which are all considered course-grain tracking).
    - In addition, the authors suggest SSL should have support for random-length padding for all cipher modes (it currently only supports it for block ciper modes), which would make it more difficult for attackers to figure out what is being sent from the length of the ciphertext sent.