1. SSL and HTTPS
   1. F. EV is more trustworthy
   2. T.
   3. T. Dane provides public key as part of the DNS response, which tells the client that the server does support TLS, and so the browser can use it to force usage of HTTPS.
   4. T.
2. Click Trajectories
   1. Pass
3. Mark Silis and Jessica Murray
   1. T
   2. T
      1. Answer: AFS did not not have hard-coded permissions for any net-18 address.
   3. F
   4. T
4. Lava
   1. Pass
5. Symbolic/ Concolic Execution
   1. Once. There are no branches in this function for our concolic execution to operate over, so once it gets to the return statement for the first time (with some value of `n`), then it will finish its test.
      1. Answer: Since the statement "2: Done" is printed upon ever execution of `f()`, since `n` is a 32-bit signed integer, we see that there are 2^31 different execution paths through f(). Lab 3 concolic execution may iterate over all possible values of `n`, though notably, lab 3 also set a `max_iter` variable of 100. So real answer is 100.
         1. Notably, all values of n that are `n <= 0` yields the same execution path, and positive values of `n` yields it own distinct execution path
            1. This tells me that iterations within while loops are indeed considered to be different execution paths (just as we saw in for loops).
         2. L
   2. It's impossible to tell. The concolic execution will likely try to some value of `n` (perhaps starting with 0) but there's no deterministic way to figure out what values it tries and how many. After 0, it is up to Z3 to choose subsequent values that may yield in different paths.
6. Web Security
   1. Yes. The attacker can register and log on the website as a sample user, inject Javascript code into the vulnerable change profile input field of the website (which doesn't sanitize input) to execute an XSS attack, which we can then inject an invisible iframe on the website that loads the login page. This will cause the login cookie to also be loaded in the iframe, of which we can then freely inject Javascript code (since both are in the same domain) that will then send or log the user's cookie to the attacker.
7. TCP Hijacking
   1. The adversary can first establish a normal connection with S as itself to discover the current sequence number, and then start another connection as C in the next second with the sequence number incremented by 64+128. We can try this many times to get a situation where S doesn't get any new connection (other than the one from the attacker) over the course of a second, which is likely since S isn't busy.
   2. Yes, it would make the attack harder, since each starting sequence number would be unique to a connection, and so we can't use information from one connection to inform us for future connections.
      1. More briefly, it's harder b/c adversary will have harder time guessing next sequence number to use. However the drawback of this is that the server has a harder time detecting duplicate connection requests. This is why TCP sequence numbers must increase slowly.
8. Secure Handshake
   1. The adversary can launch a MITM attack on the first connection that C makes to the server, where it replaces message B with its own public key and sign the message with its private key. Without the certificate, the client won't be able to detect that the public key it was given is not associated with the domain/ web server, and will blindly accept it. Subsequently, the attacker can intercept following connections that C makes with the server and decrypt the messages.
   2. The adversary can prevent the server from being able to read client's message by supplying the attacker's version of message C using its own generated key K' (and preventing the original message C from the client from reaching the server, perhaps through a DoS attack). As a result, the server will have a different symmetric key on file compared to what the client is using to encrypt.
   3. Additionally, the adversary could relay message D, without the server being able to determine whether the client really sent another copy or if this is a replay attack. This may cause the server to process msg_1 twice.
   4. This invalidates guarantee of forward secrecy. If PK_s is at some point exposed in the future, all past communications will be able to be decrypt if captured by an adversary