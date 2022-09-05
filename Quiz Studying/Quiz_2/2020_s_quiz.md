1. Max Burkhardt
   1. T
      1. Answer: F. Back of the lack of scalability in terms of security jobs
   2. F
      1. Reasoning: We need app-specific logs and statistics for network intrusion
   3. T
   4. F
      1. You don't need ML for intrusion detection for web apps
2. Keybase
   1. Skip
3. SSL/TLS
   1. The attack relies on the fact that if both sides never receive the "change cipher spec" messages they will never enable message authentication and encryption. However, if we require "change cipher spec" before Finished the adversary won't be able to make progress in the SSL protocol
      1. Why? Because with "change cipher spec" message the attacker cannot remove the auth fields from record-layer without the server noticing because the server recomputes the MAC to check for integrity of Finished.
   2. In reality ChangeCipherSpec doesn't list the cipher. It is just a signal to other party that source switched to the pending SSL state, which includes ciphers negotiated in earlier steps.   
      1. Assuming that SSL lists cipher:
      2. No. If the client uses a strong encryption cipher to start with, it would be difficult for the attacker to replace the encrypted Finished message with a weakly-encrypted version that the server will accept (the attacker needs to also decrypt Finished message first)
4. Spectre
   1. g is suitable to learn the value v, since every possible value of `v` will uniquely access a specific index within array2 (assuming `v` is nonnegative). However, a potential drawback is that small values of `v` will potentially access the same cache line, which means we won't be able to distinguish them.
   2. h is also suitable to learn the value of `v` since with hashing functions each possible value for `v` will most likely correspond to a unique hash. So to check if `v` is a specific value, we can simply hash it and check if the function h ever access array2 access at that index.
   3. Note to self: It seems like we're okay with iterating through all possible values since I guess we're doing guesses by byte so the irreversible property of hashing isn't an issue (or that performance is not a concern).
5. Messaging Security
   1. Assuming that A never uses K to send message anyone else, then yes. This is because B can verify K once it decrypts it using the MAC, and it can also be sure that K was sent by A by verifying the signature using A's public key
      1. Alternative answer: No, the protocol does not provide authenticity if A ever sends a message to someone other than B. This is because when A sends a message to some other person, call C, then they learn what K is and can use it to MAC a message of their choosing and forward on the key signature originally signed by A to B.
   2. No. Since A signs with its private key and K, the message also associates A's public key with K.
      1. Answer: Yes. This is because once B knows K, it can also impersonate A to send any message as A and produce the correct MAC (since it now knows what K is).
   3. Note to self: The trick to this question seems to be that because K is now known to everyone (e.g. like Tor), everyone else can impersonate it.
6. Web Security
   1. Since HSTS is not enabled, browsers will not automatically enforce TLS connection on subsequent visits to the website to bensbites.com. Thus when Rayden later tries to connect to bensbites.com, Noah can intercept the messages and strip away the TLS layer so that Rayden will talk to the server of HTTP rather than HTTPS. Note that Rayden himself still initiates a HTTPS connection with the bensbite.com server on behalf of Rayden.
   2. Rayden can make his website evil.com to look exactly like bensbites.com, and at the initial login page when Rayden types in his username Noah can have its web service forward that request to bensbite.com, get the username profile picture, and then embed it in evil.com. That way, Rayden will be fooled into thinking that he is accessing the right website, and his password will be captured by Noah.
   3. Yes. Assuming the database (username to nonce correlation) isn't leak, there isn't any easy way of figuring out what the image is unless we are able to observe the traffic between Noah and bensbites.com directly (which at that point also makes finding out the password trivial). 
      1. This is because **SOP** (Same Origin Policy) prevents evil.com from accessing cookies set for bensbites.com in the same browser, so evil.com cannot figure out _secimg_nonce for Noah. Valid string values are assumed to be random enough to be very difficult to guess. 
      2. Also SOP prevents evil.com from forwarding the credentials logins and reading the responses from bensbites.com
7. SUNDR
   1. No, because if there are more than 2 users making changes to the filesystem, it's possible for one users to have their modification missed by other users on the filesystem. 
      1. For example, consider situation with 4 users, (A,B,C,D). We consider user A and B to have made a number of modifications, say 10 and 11, with B making the last change. So their version structure would be something like (A:10,B:10) for A and (A:10,B:11) for B. When C makes its first change, we will see its version structure be (C:1,B:11). A and B is then able to observe C's changes and make another series of alternating changes, so their version structures only contain entries for A and B. At that point, if D looks at the file system, an adversary can just give it A's and B's latest version but give an empty entry for C. This would look okay to D, which is an issue since that results in fork consistency (since A and B knows that C made an entry).
   2. More briefly, if, say two users frequently make modifications to the FS. At some point a third user also make a change. That change would then be written out of the two users VS's as they continue to make changes, and so an adversary can remove the third user's VS structure from the SUNDR filesystem reply, which causes other users be not aware of the third user's change.