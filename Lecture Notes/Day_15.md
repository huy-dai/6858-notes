# Certificates

Recall SSL (much simplified, and certificate omitted)
```text
  [diagram]
  C is a browser, S is a web server (e.g. amazon.com).
  C --> S: connect
  C <-- S: PK_S
  C --> S: Encrypt(PK_S, freshKey)
  C <-> S: Encrypt(freshKey, data)
  From here on, ordinary HTTP/HTML inside SSL.
```

The difficult part which we'll talk about today is verifying that the `PK_S` provided is indeed owing specifically to the server and not some adversary's public key. One way we do that is through **Certificate Authorities (CA)** (another is through Kerberos).

## Certificate Authority (CA)

A CA is an organization which client trusts to validate (DNS name, public key) bindings. The CA sends back to the client the `certificate`, which is simply a signature of the CA's private key with the response information.

`cert = Sig(SK_CA, target_domain_name, PK_s)`

What's in the certificate that a website like `amazon.com` sends to the browser?
Your browser will show you -- click on the lock icon.
1. subject name -- the server's DNS name, e.g. "amazon.com"
2. subject public key
3. expiration -- typically a year or two
4. issuer ID -- CA name
5. issuer's signature over the above

## Certificate Registration

How does a website gets a certificate from the CA? A website like Amazon will generate a public/private key pair. It proceeds to send the public key and some proof of ownership (to show it actually owns the domain) to CA. CA then validates the proof of the ownership, generates the certificate, sign it with CA's private key, and sends the certificate to amazon.com

A notable step is that amazon.com can then hold on to this certificate and hand it out to its client (CA is not on the critical network path). The client can then validate a certificate from SSL as follows:
1. Make sure subject name in cert == name in URL (is the certificate actually valid for the website that I'm trying to visit?)
   1. Certificates are usually very specific (to this hostname only on this domain), though they can sometimes contain wildcard characters for more general matching.
2. CA name is recognized, and that browser knows CA's public key.
   1. This check is made possible by having the browser maintain a list of "root certificates" w/ public keys.
3. Ensure certificate is signed by CA's public key, cert has not expired or revoked

If the checks succeed, the browser will display the web page. Browsers like Chrome and Firefox will show a gray lock icon next to the URL.

If the checks don't succeed: The browser won't display the web page, and it will show the user an error, e.g. "invalid certificate".

## Man-in-the-Middle Attacks (MITM)

What are common MITM attacks and how does certificates help prevent them?

Imagine an active attacker which intercepts messages between the client and the server. Without certificates the attacker can inject their own public key instead of the actual `amazon.com` public key so that it can peer into data you send to Amazon.

However, with certificates, even if an attacker injects its own public key and supplies the actual Amazon certificate, the client will reject it because the provided public key doesn't match with what was provided in the certificate.

In addition, the attacker can produce a certificate for `attacker.com` instead of `amazon.com` along with its public key, but the client will reject it because the URL the certificate is for mismatches with what the user is trying to browser. 

Lastly, the attacker can sign its own certificate for `amazon.com`, but the client will reject it because it didn't come from a trusted CA.

## Other possible attacks

What other avenues can the attacker try? (not covered by certificates)
- Trick you into connecting to the wrong DNS name (phishing).
- Trick you into not using HTTPS at all, but rather HTTP.
  - One thing the paper mentioned is that the adversary can intentionally drop answers or return errors which will cause the client to fall back to HTTP.
  - Also what's not so clear is that while most parts of the page will use HTTPS, some parts of the page that gets loaded in (like in iframes) or sent from submission forms may be transmitted with HTTP.
- Trick a sloppy CA in to issuing attacker an amazon.com cert.
  - This is become an increasingly worrying issue given the number of CAs which has came up over the years.

## Importance of name == intent

**Case Study:** Use of TLS in 802.1x Wifi Authentication. When we connect to MIT Secure, we establish a TLS connection the authentication server. In that connection, we (the client) sends our username and password the server. If successful, we will get a certificate back. However, how do we know to *trust* the server / verify the certificate we get back is actually the one we're looking for?

It's a trick question, we don't! Our phone will just accept any certificate by default, because there's no defined relationship between the Wifi name and the domainname that is the authority says it is serving for that Wifi.

In addition, it's possible to create a rogue Wifi Access Point that poses to be "MIT Secure" and by default machines will attempt to authenticate to that AP by sending to it the username and password for authentication, because there's no way for client to know beforehand who should be a valid authentication server.

**Note:** In modern days, it's possible to explicitly specify what server names we should expect to authenticate to, e.g.  Can explicitly specify what names to expect, like `oc11-radius-wireless-1.mit.edu` or `w92-radius-wireless-1.mit.edu`.

## CA Validation Process

How does the CA validate if an entity which is trying to register a public key is who they say they are? The person making the registration need to be able to prove that they have control over the domain. In the past they would do this by sending an email to the owner of the domain (using WhoIS record or related). 

Nowadays this process has been standardized into a protocol named **ACME protocol**. ACME revolves around the idea of issuing a "challenge", where the CA issues a file over *HTTP* which it asks the domain owner to make available on their website. If the CA is able to successfully look up that challenge file under the target domain, then it approves the certificate. The certificates which uses this process for validation are called **DV (Domain Validated) certificates**.

**EV (Extended Validation) certs** are another type of cert, and is a attempt to certify both domain name ownership *and* trustworthiness. CAs are only allowed to grant EV certificate to a legitimate representative of a reputable business. In addition, the certificate will show the company name (not just domain name) and an "EV" flag. However, this attempt has shown to be not very effective since users don't seem to behave differently with or without EV. EV certificates are also expensive to issue out.

In addition, to fight against MITM attacks, the CA implements **multi-perspective validation**, where multiple CAs probe a server for the challenge text. That way, we don't rely on just one CA's perspective, which may be flawed or potentially controlled by an attacker.

## Certificate Revocation

In some cases, we want to have the ability to revoke a certificate before it is set to expire. This is useful for when, say, a data leak occurred and your pub-private key pair was exposed, or when your domain is sold to some other company. The top-level plan for CA in terms of revocation is to maintain a *revocation list*. A domain admin, upon proving their control over the domain, can ask the CA to revoke one or more of its certificates.

**Certificate Revocation List (CRL):**
1. Amazon asks the CA to revoke its certificate.
2. CA maintains a list of revoked certificate -- CRL.
3. CA server will tell anyone the CRL (and sign it).
4. Browser should fetch and check CRL as part of certificate validation.

A link to the CRL is embedded as part of each certificate that the CA issues out. Thus the browser has the responsibility to download the CRL from the CA and verify that the certificate is still valid. However, this introduces an issue with bandwidth overhead since the CA is once again on the critical network path. 

Another way to check certificate revocation t is through the **Online Certificate Status Protocol (OCSP)** where the browser sends the certificate to the CA and ask whether it is still good. The CA then checks it against its CRL list (amongst other checks) and signs the respond. This does indeed make the response smaller *but the CA still needs to sign these messages*, which cost lots of computation power.

The design that is most used now is called **OCSP stapling**. These are essentially short-lived OCSP responses signed by CA. It's the server to periodically query the CA checking on status of its certificate, and the CA will provide a response. In return the user will get back both the certificate and the OCSP response (the browser knows to expect a stapled OCSP response by having a "must staple" flag within the certificate).

## Key Pinning

The **key pinning** approach help the users and browsers detect key changes. The main idea is that we should have some mechanism in place to detect when detect certificates that we receive has a different public key than previous versions of the cert that we've received. There are multiple ways to do this:
1. Simplest pinning: 
   - Chrome has list of CAs who can issue for google.com. But this approach is not adopted by all servers across the Internet.
2. Pinning via browser history
   - Browser remembers each host's public key the first time you visit, and it wWarns you if a host's key changes in the future. This technique would correctly detect a bogus cert if you had previously used the site (the paper calls this a **Trust On First Use (TOFU)** scheme), but there are gray areas / situations when it may return a false positive e.g., what if site legitimately switches to a new key?
3. Pinning managed by web host server
   - This method uses HPKP, via Public-Key-Pins HTTP header. Server specifies its public key, or CA's public key, and a period of time the pin should last. This is a pretty good method but things can go wrong is the server doesn't properly specific these fields correctly or gives too short/long of an expiration duration.
4. Pinning via DNS
   - DANE stores the public key in DNS directly, no CA needed, or we have the identity of the CA allowed to issue certs for a domain. That way, a different corrupt CA can't issue bogus cert. However, DANE depends on DNSSEC, which is not widely used, and most browsers don't support DANE.



## CA compromise

In this whole system, you are depending on the security of the CA, which also makes it the weakest point of the system (fundamentally, the CA is inherently trusted). Sometimes, for large applications, it may be more preferable to host their own CAs (like for WhatsApp). As result, **certificate transparency** is becoming a more common industry-practice. This process requires for CA to have to talk to the **Certificate Transparency (CT)** log which maintains a global log of all certificates issued by different CAs. Thus,

1. CAs must register new certs in the CT log
2. The CT log server then returns a *receipt* which the CA keeps alongside its certificate
3. When a browser gets a certificate it will also ask for the associated receipt, otherwise it will reject the cert (however in practice browsers don't always validate or ask for receipt)
  
The nice thing here is that even if a malicious or bad certificate is issued, certificate owners can check the log for bogus certs issues in their names. Thus any bogus certificates will be found out eventually, and usually in a short-time period if certificate vendors are good about validating certs.

