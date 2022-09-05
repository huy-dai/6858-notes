1. Guest Lectures
   1. T
      1. Answer: The attack ran the server that accepts incoming email out of TCP sockets. It did not flood any network link.
   2. T
      1. Email were broken into pieces and then reassembled slightly differently for forwarding, which caused the signature to fail at the receiver.
   3. F
      1. The report was inaccurate
2. Max Burkhardt
   1. F
   2. T
      1. Mutual TLS can authenticate the client and server to each other
   3. T
      1. Network segmentation is good
   4. F
      1. airBnB has an automatic way of computing a global map based on reading configuration files of individual services
3. Tor
   1. T
      1. Tor does sacrifice some security benefits for performance and reliability. For example, it doesn't delay message or use cover traffic (to allow for greater usability and less latency).
   2. T
      1. Global adversary is a security concern for Tor.
   3. T
      1. Tor traffic has so much that it is now overburden to directory server listing due to the number of proxies available
   4. F
      1. Tor defends against such attack (one bad proxy among many)
4. Symbolic/Concolic Execution
   1. No, because we are putting on too much constraints. When wanting to explore the other side of branch b, we should only require the n branches before it to be true and for !b to be true. Any other conditions after it is considered unexplored under this new branch condition
   2. We should change cond_list to be:
      1. `cond_list = branch_conds[:b]`
      2. Note: Yes! We want to keep all constraint until *right* before b and go the other just on the bth branch. None of the branches past b are relevant in this situation. The conditional should just read i < b.
      3. Reminder: `b` is an index, not an object!
   3. The concolic system fails to find the rest of the possible results because arrays access are much more difficult to reason and represent compared to if/else statements. Thus the results that were found were done by running the solver to generate different possible inputs and test to see if it hits any new result. By this method, it's possible that we won't be able to cover the range of possible answers for function `g`.
      1. There is no support for symbolic indexing into a concrete array. 
      2. Additionally, returning the value 3 is not recognized as a branch, so this statement doesn't produce any additional path constraints. (Most important!) As a result, we only got result from indexing into array at index 0 (e.g. 3) and the base cae of -1.
5. TCP Hijacking
   1. The attacker need to suppress message B to the actual machine T, since if T receives message B, it will realize that it did not initiate a connection with the server S and attempt to reset the connection (this stopping the guessing attack).
      1. Reset connection is done with connection reset (RST) packet.
   2. The attacker can preemptively sends message C as soon as it thinks S is about to send message B. That way, message C arrives at S before T can attempt to reset the connection
      1. A better way, as mentioned in the paper, is to wait until T is down, or DoS T directly. We can also influence some switch in the middle by overflowing its queues to ensure that the message is lost.
   3. As noted, my attack still fails because the connection may still be reset before any meaningful amount of data is shared.
6. Secure Handshake
   1. No. As mentioned, the adversary still doesn't know K, and so it won't be able to create a create a valid encrypted message and corresponding MAC that S will accept.
   2. No. With this protocol all ephemeral relating to the current conversation is discarded at the end, so both sides doesn't have any persistent information from the secure channel that can allow them to be sure of the other's identity in subsequent conversations. Cookies can provide a method for establishing identity of C from previous connections, but this not part of the protocol
      1. C never signs any message with a private key and S cannot rely on the source IP address.
   3. It is difficult (look at the wording!). This is because even if the attacker knows SK_s they will also have to be able to guess SK_t to figure out what K is to decrypt messages since it was deleted afterwards, which is not possible.
7. Web Security
   1. Because the website doesn't sanitize the user inputs (say to view balance of a user), we can perform a XSS attack where we inject malicious Javascript script code into the page that can perform operations such as transferring bits to our account.
      1. Cross-site scripting allows you to run code in the same origin as the target, which circumvents SOP. Cross-site request forgery allows requests to be posted to a different origin, and while the attacker cannot read the response, they can cause the request to be issued.
   2. This attack will work, because the user was tricked into actually clicking the transfer form when they didn't intend to. Same-origin policy doesn't prevent this attack because the form was indeed submitted directly from the original web service; it was simply that the user was "tricked" into clicking something they did not mean to.
      1. Since the form is the true form loaded on the website, any CSRF tokens added will also be sent alongside it. Also note that Alice site does not contain a form, so it doesn't need the cookie or any CSRF tokens.
   3. **Same Origin Policy** does not prevent against writes, only reads! Thus before this change, CSRF attacks are still an issue for Ben's website because the attacker can inject some hidden image on their own website with a href link that automatically sends a request to Ben's bank website to transfer bits. By default the browser will also send any relevant cookie alongside the request. With this change, this is no longer the case and so the attacker can't steal bits from the user.
      1. Note: Attacks within Bank's own origin are not defeated by this header. Clickjacking is also not prevented.
   4. Side Channels
      1. No. Spectre will not work without speculative execution because it relies on training the branch predictor to go down a branch that is not taken. 
         1. The attacker won't be able to get in branch to be executed with x that cases access into array1 to reach the secret.s
      2. Yes, we just need to make sure that array2 is never loaded in the processor cache prior to execution, and/or make sure to fill the processor cache with other pieces of data.
         1. Note the instruction! The attacker doesn't have a way to evict the cache indirectly. 
         2. In that case, no, it's not possible because whenever we check if an entry from array2 is in the cache we won't be able to tell which ones have been accessed by `victim_function` and which ones were already there.
      3. It will hinder our ability to identify the secret, since it's possible any load for array2 done by the speculative execution will be overwritten / evicted from the cache by the memory-intensive program before we can detect it. Thus we'll need to run the program over many cycles to be sure we're able to find the secret.
      4. No, because array1 resides in kernel memory, so outside of the speculative execution we won't be able to access it to make cache line measurements. This is why we need array2, which we control and can readily access for the second part of the attack.
         1. Answer: No, because in this case the value we're after `temp` is not being used in our measurements. If we're measuring the presence/absence of cache lines in `array1` only reveals the value of x, and not the secret.
8. Untrusted Version Server
   1. T/F
      1. T
      2. F
      3. T
   2. No, they must be different, because for both users to properly validate the log response, in Bob's update, he must see that the previous hash in Angry Turtle v1 log record points to SecureChat v3 (since that's included as part of the reply), while Alice must see the previous hash pointing to SecureChat v2.
      1. Since the logs that the server gives Alice and Bob differ, the hashes that they see in the Angry Turtle log records must differ too.
      2. At first I was confused on how this was possible, but as mentioned the Version Server may be malicious (also in the name of the problem).
   3. T/F
      1. F. New updates may have come in between requests
      2. F. Server may be hiding records from both users.
      3. T. fetch() result passes and tail is correct.
      4. T.
         1. This is a useful property to quickly check if the server is dishonestly showing them different information.
