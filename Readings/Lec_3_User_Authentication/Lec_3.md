## Your password doesn't matter

Link: <https://techcommunity.microsoft.com/t5/azure-active-directory-identity/your-pa-word-doesn-t-matter/ba-p/731984>

Password sprays, where the attacker attempt the same set of passwords over a known user list against the online service, is relatively ineffective as long as your password isn't in the top 50 of most commonly used.

However, if you consider the case where the database gets extracted and the attacker can run against hashes offline, the likelihood of your account getting cracked is immensely higher. Nowadays you can build cracking rigs that does like 100B hashes per second, and using some subset of the >500M known passwords we can get some really high percentage of passwords getting cracked (> 70%). Plus cracking rigs can take into account the policy or common structures [Capital letters, lowercase letters, numbers, symbols] and you can increase the effectiveness of the cracking.

The punchline is 2FA should be enforced nearly everywhere!

Also, some ways that AD has tried to make cracking harder is with storing salted hashes after _1000_ runs of SHA512, which increases computation time.

## Security Keys: Practical Crypto Second Factors for Modern Web

Who: Researchers from Google

### Current Methods and Drawbacks:

* OTPs - 6 to 8 digit codes sent to user via SMS or generated by physical dongles. 
    * OTP (one time passcode) are vulnerable to attacks like phishing, MITM, and have usability drawbacks (e.g. need smartphone and data availability).
* Smartphone as Second Factor - While promising, protecting application logic from malware is difficult on a general purpose computing platform.
* TLS client authentication - One way to implement is through certificates. 
  * However, as currently implementation, these yield in a poor user experience. also, because client certificates are transmitted in the clear, the user's identity is revealed during transfer to any netwokrk adversary. 
  * TLS client certs are hard to move
* Other methods, like national ID cards and smart cards require custom reader hardware and sometimes can make it harder for users to protect their privacy

### Threat Model:

Malicious websites, reusing cracked credentials, network attackers (sniffers, MITM), malware

**Attack consequences:** Session duplication (allow further access at any time from any computer), session riding (can only access account when user is actively using their computer)

## Design

Security Keys supports _registration_ (generating new key pairs and returns public key to be associated with user account), and _authentication_ (answering to challenge using private key)

Note that the Security Key upon generation associates itself with a specific web origin, and so when it is asked to authenticate to a challenge, the Security Key will refuse any request that doesn't fit the expected signature. 

In authentication, the Security Key will also sign into the challenge a counter to indicate the number of signature that the key has performed (to help fight against cloning of keys). The client data also binds the TLS channel ID to it to allow the server to detect the presence of a TLS Man in the Middle.

Security Keys will require some form of confirmation that a human is performing the authentication, e.g. through a capacitive touch sensor or mechanical button.

The storing and retrieve operations done by a Security Key can be thought of database operations. To prevent an attacker from quickly identifying the number of accounts a Security Key is used with, these operations are done as a key wrapping/ unwrapping operation.

**Security:** Security Keys do not depend on a trusted third party. They also argue that Security keys generate assertions against phishing and website attackers, since it generate unique key pairs per account and restrict key use to a single orign.

