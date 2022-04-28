## Day 3 - User Authentication

From last lecture, we discussed the idea of having a **guard** that takes in requests from a principal and decides on whether/ how they want to act on it.

## Parts of an Authentication System

These are functionalities that are required for auth system:

1. Register - Add user to the database; make the guard aware of the user
2. Authenticate - Verify that user credentials matches
3. Recovery - Provide option for user to reset their credentials if they lost it
   1. The specific solution will need to be tailored to the system we're working with. However, this is almost certainly a required functionality since it's not a very useful system if credentials never changes or we don't have a backup auth option.

### Challenge 1: Intermediate principals

Betweeen the user and the actual guard, there are many different layers. e.g. user -> laptop -> browser -> web server -> guard. Malware can intercept your request at each of these component layer, which may negatively impact 

### Challenge 2: Identity

How do we verify that the user is who they say they are.

For example,

* Bank: Name, DOB
* MIT: admission
* Many systems provide a weaker guarantee, which is "step 2 (authentication) matches step 1 (original credentials)". They really don't know anything else about you.
  * These system are also built on the premise that the services know less about the users.

## Implementation

### Registration

- First-come first-serve: email@gmail 
  - Once it's taken no one else can register an account with it
- Boostrap from other system
  - Relying on some other service to connect with your identity. For example, Amazon sends a verification e-mail to your email address - e.g. if you have access to this e-mail you should be able to create a principal with it
    - Also provides a mean of account recovery
- Administration creation 
  - Accounts managed by some broader administrator

### Recovery

- Security Questions: "OR"
  - They are meant to be easy so that users don't have to remember explicit response but 
- Recovery Email
- Ask admin (like IS&T) or customer service
  - Can be subject to social engineering attack. Need to have a strict process in place to properly verify identity
- Use other ways to log in, if available
- Sometimes the answer is no.
  - You tell your user to just create a new account
  - Occurs if there is no way for the service to verify your identity due to lack of info

## Passwords (Authentication Method)

**Stats:** 5,000 unique passwords account for 20% of users (6.4 millions). Similar statistics confirms this again in 2010 (Gawker break-in).

When we're talking about passwords, we're often talking about authentication over the network, which is harder to place rate limiting on compared to local logins like pasword managers. For example, the bad guys can also intentially just lock you out of your account.

**Discussion:** Format vs. entropy
    - The bad guys are very aware of the distribution of characters in a password, which they can use to their advantage when generating cracking attempts
    - Thus it's very important to have high entropy in your passwords to make these format techniques less usefuls

Also, users often share the same password across platforms.

**Question:** Are fingerprint readers and facial recognition better than passwords?
  - Privacy issues / concerns
  - Biometrics are best thought as an identity rather than an authentication system. The problem is that they can be vulnerable to replay attacks and also if your fingerprint hash or ID is known or exploited at some point it's no longer viable as a login option
  - However, the cases where biometrics work well is if you are authenticating directly to a device (like FaceID to IOS devices), since for the attacker to replicate this they will need:
    - To have your device, and
    - To have your face or somehow fool the recognition system
    - Note: It might be possible to extend this model if we say the device is a trusted Finger Print authenticator and so once the user authenticates with the device the device will be able to authenticate the user with the server directly.

**Storage:** Instead of storing raw passwords, store them as a hash instead. However, with this we are worried about pre-computation attacks (like rainbow table). Thus, it's important for us to add salt (and pepper) for each user hashes. 

Also, to increase computation time for each stored hash, we can go with other hash algorithms like PBKDF2, Bycrypt, etc. which takes on order of 1 second to run. This additional time is not very distinguishable to the user but makes it hard for attackers to run large-scale brute force attacks. Similarly, we can run the same hashing algorithms many times (like SHA256 a thousand times)

## 2FA

- SMS-based authentication: Server, upon receiving proper user credentials & verifying it, then pulls the associated phone number with the user and sends a newly-generated code to their phone. The user sees the code, enters it, and the server verifies it is the same.
  - **Pros**: Can enforce TTL for code and rate-limiting. Also it is relatively easy to implement
  - **Cons**: SMS messages are not secure. There many serious attacks that have occured over the past with SMS spoofing and SMS swap. Cell networks can be hacked, and there exists many insecure cell services that you can sign up and receive other people's phone messages. Also users may be vulnerable to phishing attacks. 
- Time-based one-time password (TOTP): Instead of the server sending the code to the user, they are deterministically computed using some combination of password hash and current time (and potentially other elements like some recovery integer value). Both the server and the authenticator app on the user's phone has access to all the information they need to compute the code.
  - **Pros:** No need to send auth code over untrusted cell network
  - **Cons:** User will need to handle recovery themselves. If they lose their phone it is very hard to get back access. Also it's still possible to phish the users (e.g. posing fake website as real)
- U2F:
  - We have three actors - The device containing the Security Key, browser, and the service
  - **Functions:**
    - The device stores the public (K_pub) and private (K_priv) key which is generated by Security Key for every authentication use case. On the other hand, the server, upon registration, gets K_pub.
    - The Security Key is able to create a signature using K_priv which it adds to its challenge response, which then is verified by the server using the given public key.
  - **Process:**
    - The server generates a random number (called a "nonce" since it's used once). The number is relatively large, like 64-bit or 128-bit. It sends the number as a challenge to the Security Key through the browser. In a simplified world, the browser forwards it to the Security Key, which signs it. That signature is sent to the server, and the server verifies that it properly wraps the challenge that it asked for.
    - In the real world the K_pub and K_priv is generated on the fly, and all the storage is done on the server. 
    - Also there's additional work done in registration stage where the Security Key associates the new key pair with the browser's web origin through a key handle, which helps the Security Key itself verify the browser's legitness later on in the authentication step
      - This feature fights against phishing attacks, the adversary pretends to be a real server, and through sending known challenges they can derive your actual signature. The same signature can potentially be reused for different sites.
      - Also U2F attempts to fight against replay attacks by making the Security Key sign the domain name it thinks it's responding to in the challenge as part of the signature. Thus if a fake website is sitting between user and actual service, when the real server receives the packet it will be able to detect the user has been misled to respond to a fake service.
  - **Pros:**
    - Can't reuse the signature multiple times
    - Allow for dynamic key generation
  - **Cons:**
    - Now that U2F has been put into practice, some elements of the original protocol has discovered to be less useful or secure as intended, most notably the counter ID and also the attestation form.
