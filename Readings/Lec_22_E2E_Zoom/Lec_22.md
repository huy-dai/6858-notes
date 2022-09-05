# E2E Encryption for Zoom

Link to [research paper](https://css.csail.mit.edu/6.858/2022/readings/zoom_e2e_v3_2.pdf)

##  Encryption

When a Zoom client gains entry to a Zoom meeting, it gets a 256-bit per-meeting key created by Zoom server, which retains the key to distribute it to participants as they join. This per-meeting key is used to derive a per-stream key (by combining the per-meeting key with a non-secret stream ID using an HMAC function). Each stream key is then used to encrypt audio/video (UDP) packets using AES in GCM mode, with each client emitting one or more uniquely-identified streams. 

Those packets are then relayed and multiplexed via one or more Multimedia Routers (MMR) in Zoom's infrastructure. The MMR servers do not decrypt the packets to route them. Additionally, by taking advantage of the fact that **Zoom servers do not require any access to meeting content**, they can ensure end-to-end security at a large scale. However, notably, the current design **does not provide end-to-end key management.** This means an adversary who can monitor Zoom's server infrastructure and access to relevant Zoom servers may be able to defaut encryption.

## Threat Model

They define adversaries to include outsiders, meeting participants, and insiders. In particular, to seek to provide the following security guarantees:

* Confidentiality: Only authorized meeting participants should be able to access meeting audio and video streams
* Integrity: Those not allowed into a meeting should not be able to corrupt content of meeting
* Abuse Prevention: Zoom provides an effective mechanism to prevent abuse from authorized meeting participants + reporting.

### Things not in threat model

1. Metadata and traffic analysis: Even for E2E encrypted meetings, insiders and outsiders can learn details about meeting duration, meeting bandwidth, data streaming patterns, participant lists and IP addresses
2. Zoom sign ons: Zoom leverage third-party Single Sign-Ons (SSOs) and Identity Providers (IDPs) to independently vouch for the identity of Zoom's users. This moves the trust away from Zoom and on the individual identity providers.

## Security Roadmap

They provide a four-phrase roadmap for the long-term security of Zoom

### Phase I: Client Key Management

In the first phase, they will roll out public key management, where every Zoom application generates and manages its own long-lived public/private key pairs. From there, they will upgrade session key negotiation so that the clients and generate and exchange session keys without needing to trust the server. 

With the "E2E" security feature turned on, all participating clients must run on official Zoom client software (thus preventing log-ins from web browser or legacy Zoom-enabled devices).

### Phase II: Identity

In the first phase, clients trust Zoom to accurately map usernames to public keys. In phase II, they plan to introduce two parallel mechanism for users to track each other's identities without trusting Zoom's servers. The first involves allowing SSO IDP to sign a binding between Zoom public key to an SSO identity, and to push this identity through the UI (thus preventing Zoom from faking this identity). The second feature is to allow users to track contacts' keys across meetings, so the UI can provide warnings for when a user joins a meeting with a new public key.

### Phase III: Transparency Tree

They plan to implement a mechanism that forces Zoom servers to sign and immutably store any keys that Zoom claims belong to a specific user, forcing Zoom to provide a consistent reply to all clients about these claims. Clients will then periodically audit the keys that are being advertised for their account and show any unexpected new additions to the user. Additionally, audit systems and routinely verify and raise alerts for inconsistencies they find.

During the third phase they also want to allow meeting leaders to "upgrade" a meeting's E2E encryption once it began.

### Phase IV: Real-time security

In this phase they look to further lock down security when it comes to signing in with new devices, through using an SSO IDP to reinforce device additions or delegate such tasks to the IT manager.