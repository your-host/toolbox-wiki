<img align="right" src="images/logo-internetnl-en.svg">

# DANE for SMTP how-to
This how-to is created by the Dutch Internet Standards Platform (the organization behind [Internet.nl](https://internet.nl)) in cooperation with industry experts (hosters and vendors) and is meant to provide practical information and guidance on implementing DANE for SMTP.

👉 Great, now just [show me how](#generate-dane-tlsa-record-with-example)

# Executive Summary
* DANE is a best-practice technology for securing the transfer of email (SMTP) between organizations across the public Internet.
* Successful DANE deployments require additional operational discipline.
    - Automated monitoring of your own email servers and related DNS records is a must.
    - Robust automation of coordinated DNS and email server certificate chain updates.
    - These topics will be covered in more detail in this how-to.
* Please plan carefully and then deploy DANE for your email servers. Botched deployments not not only harm the domain in question, but also have a deterrent effect on adoption by others.
* For more information on adoption statistics and software support, take a look at: [https://github.com/baknu/DANE-for-SMTP/wiki](https://github.com/baknu/DANE-for-SMTP/wiki)

# Table of contents
- [What is DANE?](#what-is-dane-)
- [Why use DANE for SMTP?](#why-use-dane-for-smtp-)
  * [Risks of SMTP with opportunistic TLS](#risks-of-smtp-with-opportunistic-tls)
  * [DANE addresses these risks](#dane-addresses-these-risks)
  * [How about MTA-STS?](#how-about-mta-sts)
- [Generate a DANE TLSA record for inbound e-mail traffic (publishing DANE DNS records)](#generate-dane-tlsa-record-with-example)
- [Advantages of DANE explained by illustrations](#advantages-of-dane-explained-by-illustrations)
  * [Mail delivery: TLS without DANE](#mail-delivery--tls-without-dane)
  * [Mail delivery: TLS with MITM stripping TLS](#mail-delivery--tls-with-mitm-stripping-tls)
  * [Mail delivery: TLS with MITM using evil certificate](#mail-delivery--tls-with-mitm-using-evil-certificate)
  * [Mail delivery: TLS with DANE](#mail-delivery--tls-with-dane)
  * [Mail delivery: TLS with DANE without DNSSEC](#mail-delivery--tls-with-dane-without-dnssec)
- [Reliable certificate rollover](#reliable-certificate-rollover)
  * [Points of attention when rolling over using "current + next"](#points-of-attention-when-rolling-over-using--current---next-)
- [Tips, tricks and notices for implementation](#tips--tricks-and-notices-for-implementation)
- [Implementation guides:](#implementations)
  * [Postfix](DANE-for-SMTP-how-to-Postfix.md)
  * [Exim](DANE-for-SMTP-how-to-Exim.md)
  * [Halon](DANE-for-SMTP-how-to-Halon.md)
  * [PowerMTA on port 25 (outbound only)](DANE-for-SMTP-how-to-PowerMTA.md)
- [Special thanks](#special-thanks)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# What is DANE?
DANE is short for "**D**NS-based **A**uthentication of **N**amed **E**ntities" and is described in [RFC 6698](https://tools.ietf.org/html/rfc6698) with "updates and operational guidance" in [RFC 7671](https://tools.ietf.org/html/rfc7671). DANE establishes a downgrade-resistant method to verify an SMTP servers identity **before** it starts to transport an email message over a STARTTLS encrypted layer. In order to achieve this it uses verification information retrieved from the recipients DNSSEC signed DNS zone. DANE does not rely on additional trusted parties outside the delegation chain in DNS. 

DANE, as a method, has been designed to work with any TLS service. DANE for SMTP (which is described in [RFC 7672](https://tools.ietf.org/html/rfc7672)) implements the DANE method for SMTP. It is used increasingly and adds active attack (man-in-the-middle) resistance to SMTP transport encryption [RFC 7672 Section 1.3](https://tools.ietf.org/rfc7672#section-1.3). DANE for SMTP uses the presence of DNS TLSA ressource records to **securely signal TLS support** and to publish the means by which SMTP clients can successfully **authenticate legitimate SMTP servers**. The result is called "opportunistic DANE TLS", and resists downgrade and man-in-the-middle (MITM) attacks when the destination domain and its MX hosts are DNSSEC signed, and TLSA records are published for each MX host. While possible, DANE for HTTP is not presently supported by the major browsers and so has seen little deployment.

# Why use DANE for SMTP?
At this time it is not possible to require **mandatory** transport encryption (TLS) in public mail transport. A mail server might not support transporting messages using encryption. Today only plaintext or **opportunistic** transport encryption are applicable – opportunistic because it is up to the receiving server to decide if it wants to and is able to send messages using TLS (via STARTTLS).

DANE offers several advantages by binding X.509 certificates to domains using DNSSEC. In an SMTP context this allows **a)** sending mail servers to securely signal TLS support by the receiving domain's mail server, and **b)** sending mail servers to verify the autenticity of the offered certificate by the receiving domain's mail server. This helps to address risks that occur when using oppurtunistic TLS connections between mail servers.  

## Risks of SMTP with opportunistic TLS
The way SMTP was designed including the usage of opportunistic TLS (via STARTTLS) is not without risks:
* A sending server has no means to determine **beforehand** if the receiving server supports encrypted transport. Only after the communication has begun the sending server may learn from the features the receiving server supports that it allows for encrypted transport.
  * SMTP uses opportunistic TLS (via STARTTLS) as a mechanism to enable secure transport between mail servers. However, the fact that STARTTLS is opportunistic means that the initial connection from one mail server to another always starts unencrypted making it vulnerable to man in the middle attacks. If a mail server does not offer the 'STARTTLS capability' during the SMTP handshake (because it was stripped by an attacker), transport of mail occurs over an unencrypted connection. 
  * Forcing the use of TLS for all mail servers would break backwards compatibility and is therefore not considered a viable option since it could seriously disrupt mail delivery across the internet.
* By default mail servers do not validate the authenticity of another mail server's certificate; any random certificate is accepted (see [RFC 3207](https://tools.ietf.org/html/rfc3207)).
    - It was unclear which CAs to trust when validating the certificate for a given destination.
    - In MTA-to-MTA SMTP, server hostnames for the destination domain are obtained indirectly via DNS MX lookups, but, without DNSSEC, these names cannot be trusted.  As a result, it was unclear which names to verify in the certificate.

As as result, even when STARTTLS is used, a man in the middle attacker:
* can strip the STARTTLS capability send by the receiving mail server because this communication takes place without the use of encryption. 
* can insert any certificate of his choice in the in the unencrypted communication and use it to intercept the traffic.

## DANE addresses these risks
The risks of SMTP with opportunistic TLS can be mitigated by using DANE:
* The presence of a DNS TLSA record for the receiving domain's mail server, should be interpreted by the sending mail server as the capability to use TLS. The use of TLS can therefore be forced when communicating with the receiving domain's mail server.
* The operator of the receiving domain's mail server is obligated to ensure that any published TLSA records at all times match the server's certificate chain, and that STARTTLS is enabled and working. This makes it possible for the sending domain's mail server to validate the certificate offered by the receiving domain's mail server.

In short: DANE allows sending mail servers to unconditionally require STARTTLS with a matching certificate chain. Otherwise, the sending mail server aborts the connection and tries another server or defers the message. Receiving servers with published TLSA records, are therefore no longer vulnerable to the afore mentioned man in the middle attacks.

## How about MTA-STS?
Internet.nl currently does not include MTA-STS in its tests. This paragraph explains why this is the case. 

First you need to understand that, as [explained on our website](https://en.internet.nl/faqs/report/), the selection and development of tests performed by internet.nl is primairily based on:
* the [‘comply-or-explain’ list](https://www.forumstandaardisatie.nl/open-standaarden) of the Dutch Standardisation Forum which is mandatory for all Dutch government agencies;
* security advices of the Dutch NCSC;
* relevant RFC’s of IETF.

Final decisions in this area are made by our steering committee.

MTA-STS is a fairly new standard for enforcing authenticated transport encryption between mail servers. The standard was developed primarily by the big cloud mail providers and seems to especially meet their specific needs. We have seen [quite some uptake of DANE](https://github.com/baknu/DANE-for-SMTP/wiki/4.-Adoption-statistics) (e.g. by Comcast, Protonmail and Cisco), but until now not by the major cloud mail providers. MTA-STS is relatively complex because it needs an extra HTTPS interface (including certificate validation) to function, and it is also considered to be less secure than DANE due to trust on first use and caching. 

MTA-STS relies on trust on first use (TOFU) and policy caching. So the initial connection is just STARTTLS (without authentication of the receiving mail server and without enforcing encryption). During that first connection the MTA-STS policy will be cached by the sending mail server and subsequent encrypted connections will be enforced and authenticated (until the policy cache time expires). This caching mechanism means the larger and more up up-to-date the cache, the better MTA-STS will work. Basically big (cloud) mail providers (that contact about every mail server worldwide each day) will have a large up-to-date policy cache, but smaller mail providers will not and therefore will run more often into trust on first use (i.e. initial less secure connection). So basically MTA-STS seems to offer better security to big players than to the smaller ones. In fact, the MTA-STS RFC [states](https://tools.ietf.org/html/rfc8461#section-10) that DANE offers better downgrade resistance. 

In view of the foregoing and considering the facts that the Dutch NCSC [advises](https://www.ncsc.nl/documenten/factsheets/2019/juni/01/factsheet-beveilig-verbindingen-van-mailservers) to use DANE for securing mail transport, and DNSSEC adoption in the Netherlands (and some other countries like .SE, .NO and .CZ) is quite high and keeps increasing, DANE and not MTA-STS is on the 'comply or explain' list and subsequently included on internet.nl.

Note that MTA-STA and DANE can co-exists next to each other. They intentionally do not interfere.

# Generate DANE TLSA record (with example)
This part of the how-to describes the steps that should be taken with regard to your inbound e-mail traffic, which primairily involves publishing DANE DNS records. This enables other parties to use DANE for validating the certificates offered by your e-mail servers.
![](images/DANE-example-TLSA-record.png)

In summary, you want to use the following pattern: DANE-EE(3), SPKI(1), SHA-256(1)
```
_25._tcp.mail.example.nl. 300 IN TLSA 3 1 1 <your SHA2-256 hash>
```
Replace the value of ```<your SHA2-256 hash>``` with the value of your own certificate.
<br> Replace ```mail.example.nl``` with your (sub)domain. Multiple mailservers on the same domain need a TLSA record for each of the subdomains.

You can generate the DANE SHA2-256 hash of the public key with the following command:
```bash
echo $(openssl x509 -in "/path/to/cert.pem" -noout -pubkey \
| openssl pkey -pubin -outform DER \
| openssl sha256 -binary \
| hexdump -ve '/1 "%02x"')
```
In the command above, replace ```/path/to/cert.pem``` with the location of your domain certificate.

## Fields overview:

...moved

## Use DANE-TA if you are a SaaS-provider.

Hosting providers and mass virtual hosting ***should not*** use DANE-TA because their clients own their own domain and issue their own certificates.

SaaS-providers issue the certificate for a client (sub)domain. In that case the client would CNAME to a custom or global (sub)domain of the SaaS-provider. By using DANE-TA the provider does not need thousands of TLSA-records on their primary domain. If you are a SaaS-provider you want to use ***your own*** intermediate certificate for the TLSA record (the one closest to the issued end-certificate for your clients domain).

You will need to make sure that you include the TA certificate as part of the certificate chain presented in the TLS handshake server certificate message. Even when it is a self-signed root certificate. Also, in this case you should use the "Full certificate(0)" selector in stead of the "SPKI(1)". Such TLSA records are associated with the whole trust anchor certificate, not just with the trust anchor public key. Otherwise this may, for example, allow a subsidiary CA to issue a chain that violates the trust anchor's path length or name constraints (read more on [RFC7672](https://datatracker.ietf.org/doc/html/rfc7672#section-3.1.2)). This means using: "...TLSA 2 0 1 ..." and optionally add "...TLSA 2 0 2 ..." (if required).

You can generate the DANE SHA2-256 hash of the full certificate with the following command:
```bash
echo $(openssl x509 -in "/path/to/ca.pem" -outform DER \
| openssl sha256 -binary \
| hexdump -ve '/1 "%02x"')
```
In the command above, replace ```/path/to/ca.pem``` with the location of your intermediate ~~(or root)~~ certificate.

# Advantages of DANE explained by illustrations
## Mail delivery: TLS without DANE
The illustration below shows two TLS capable mail servers without using DANE. This scenario exposes the mail transport to the risks described above.

![](images/dane-example-1-no-dane.png)

## Mail delivery: TLS with MITM stripping TLS 
The illustration below shows what happens when an attacker performs a man in the middle (MITM) attack and forces an unsecure connection by stripping the TLS capability from the receiving e-mail server. 

* When an attacker controls the network communication between the sending and receiving server it may downgrade the session by removing the information that indicates the receiving server supports encrypted transport. The sending server would proceed and transport the message unencrypted making all data contained in the message visible to the attacker.

![](images/dane-example-1-striptls.png)

## Mail delivery: TLS with MITM using evil certificate 
The illustration below shows what happens when an attacker performs a man in the middle (MITM) attack and inserts its own certificate into the connection process.

* Traditional SMTP protocol design does not offer any means for a sending server to automatically verify it communicates with the right server. As a result an attacker may fool the sending server to hand over the message to a server under the attackers control. This is called a man-in-the-middle attack, because the attacker places the bad server between the sending and the good receiving server.
* To accieve this the attacker would configure the bad server to identify as the good receiving server. It would even use a TLS certificate that identifies with the name of the good server. The attacker would then poison the sending servers DNS telling it to go to the bad server whenever it wants to send messages to the good one. The fooled server would hand over all messages to the man-in-the-middle server.

![](images/dane-example-1-evilcert.png)

## Mail delivery: TLS with DANE
The illustration below shows how the use of DANE can protect against man in the middle (MITM) attacks by addressing the shortcomings of TLS without DANE.

* The operator of the receiving server publishes a TLSA record in its DNSSEC signed zone. The mere existence of the record indicates to the sending server that the recieving server **must** offer STARTTLS. Otherwise transport must not take place.
* The receiving server's TLSA record contains policy information and a string that identifies the correct certificate. Only the good servers certiciate will match these verification criteria. Unless the attacker is also in possesion of the good servers certificate it cannot disguise as good and the man-in-the-middle server will be detected. It the verification data does not match the servers certificate the sending server will not transport the message.

![](images/dane-example-1-with-dane.png)

## Mail delivery: TLS with DANE without DNSSEC
Although guaranteeing reliable DNS resolving is actually an advantage of DNSSEC, it is still worth mentioning here. Notice that in the example above (TLS with DANE) the lack of DNSSEC would make it possible for an attacker to alter DNS responses (2 and 4). Such an attack can be used to trick the sender into sending e-mail to a rogue e-mail server. 

# Reliable certificate rollover
A TLSA record identifies a certificate. If the certificate is exchanged for a new one also the associated TLSA record needs to be updated. Otherwise there will be a mismatch, verification will fail and DANE aware servers will not send messages to the receivers domain.

Usually this creates the intent not to exchange the certificate at all. But this collides with best-practices to replace crytographic material – certificates and keys – from time to time.
  
To satisfy both requirements establish these maintenance procedures:
* DANE allows to publish multiple TLSA records. As long as one matches the receving servers certificates everything is fine. Publish **two** sets of TLSA records like this:
  * One TLSA record matching the certificate currently in use.
  * A second TLSA record matching a fallback or future certificate to be used.
* Wait two rounds of **TTL** before you start using a new certicate. This allows clients to clear their DNS cache and pick up the new TLSA record.
* Once the new certificate has been deployed, tested and all is well, the legacy TLSA records may be dropped. Notice that removing the legacy TLSA record too soon can cause an outage. 

Two ways of handling certificate rollover are known to work well, in combination with automated monitoring to ensure that the TLSA records and certificates are always current and correct.

1. **Current + next**. This roll-over scheme always publishes two TLSA records per server certificate.
    - One with the SHA2-256 fingerprint of the mail server's current public key (a "3 1 1" record).
    - And a second with the SHA2-256 fingerprint of the mail server's next public key (also a "3 1 1" record).

## Points of attention when rolling over using "current + next"
* With the "current + next" approach, the second key pair (of which the public key can be used for the **next** TLSA "3 1 1" fingerprint) can be created / known in advance of obtaining the corresponding certificate for *that* key. In particular, if keys are rotated often enough (every 30 to 90 days or so), the next key can be pre-generated as soon as the previous key and certificate are deployed. This allows plenty of time to publish the corresponding **next** "3 1 1" TLSA record to replace the legacy record for the decommissioned key. The process begins again with another "next" key generated right away.
* Deployment of a new certificate and key must be predicated (automated check) on the corresponding TLSA "3 1 1" record being in place for some time, not only on the primary DNS nameserver, but also on all secondary nameservers. Explicit queries against all the servers are to check for this are highly recommended.
* Some servers have keys and certificates for multiple public key algorithms (e.g. both RSA and ECDSA). In that case, not all clients will negotiate the same algorithm and see the same key. This means that a single "3 1 1" record cannot match the server's currently deployed certificate chains. Consequently, for such servers the "3 1 1" current + "3 1 1" next TLSA records need to be provisioned separately for each algorithm. Failure to do that can result in hard to debug connectivity problems with some sending systems and not others.
* Use of the same key (and perhaps wildcard certificate) across all of a domain's SMTP servers (all MX hosts) is **not** recommended. Such keys and certificates tend to be rotated across all the servers at the same time, and any deployment mistakes then lead to an outage of inbound email. Large sites with proper monitoring and carefully designed and automated rollover processes can make wildcard certificates work, but if in doubt, don't overestimate your team's ability to execute this flawlessly.
* When monitoring your systems, test every IPv4 and IPv6 address of each MX host, not all clients will be able connect to every address, and none should encounter incorrect TLSA records, neglected certificates, or even non-working STARTTLS. Also test each public key algorithm, or stick to just one. All DANE-capable SMTP servers support both RSA and ECDSA P-256, so you can pick just RSA (2048-bit key) or ECDSA (P-256).
* Make sure that your servers support TLS 1.2, and offer STARTTLS to all clients, even those that have not sent you email in the past. Denying STARTTLS to clients with no IP "reputation" would lock them out indefinitely if they support DANE, since they then can never send any initial mail in the clear to establish their reputation.

# Tips, tricks and notices for implementation
This section describes several pionts for attention when implementing DANE for SMTP. 

* DANE is currently only used for transfer between MTAs (thus the MX domain). So if you are using another domain's mail servers, make sure to ask the administrator of that domain (your mail provider) to support DANE by setting up a TLSA record.
* When implementing DANE we advise to first publish DANE records on your MX domains, and then enable DANE verification on your sending mail servers.
* DANE is backwards compatible. So if your mail server supports DANE and a connecting mail server does not support it yet, usually STARTTLS or plain text is used.
* DANE relies on the security that is provided by DNSSEC. Make your primary domain and MX domain support DNSSEC before implementing DANE. It is important that DNSSEC is implemented properly. A lot of DANE breakage stems from receiving/recipient domains with broken DNSSEC implementations.
* Purchasing of expensive certificates for mail servers has no to little added value for the confidentiality since mail servers don't validate certificates by default. Depending on the context there can be other advantages which makes organizations decide to use specific certificates.
  * It is possible to use self-signed certificates. 
  * [Section 3.2 of RFC 7672](https://tools.ietf.org/html/rfc7672#section-3.2) states that SMTP clients **must not** perform certificate name checks when using an end-entity certificate (usage type 3). However, it also states that SMTP clients **must** perform certificate name checks when using an intermediate or root certificate (usage type 2). The latter is necessary to prevent attackers from abusing a random intermediate or root certificate to falsely validate their evil mail servers.  
  * [Section 3.1 of RFC 7672](https://tools.ietf.org/html/rfc7672#section-3.1) states that the expiration date of the end-entity certificate MUST be ignored.
* Make sure the TTL (time-to-live) of your TLSA records is not too high. This makes it possible to apply changes relatively fast in case of problems. A TTL between 30 minutes (1800) and 1 hour (3600) is recommended.
* The refresh value of your full DNS zone should be in accordance with the TTL setting of your TLSA record, to make sure all name servers give the same information when (after expiration of the TLSA TTL) being queried.
* Roll-over scheme "current + next" gives less flexibility but the highest form of certainty, because of "tight pinning".
* Implement monitoring of your DANE records to be able to detect problems as soon as possible. 
* Make sure your implementation supports the usage of a CNAME in your MX record. There are some inconsistencies between multiple RFC's. According to [RFC 2181](https://tools.ietf.org/html/rfc2181#section-10.3) a CNAME in MX records is not allowed, while [RFC 7671](https://tools.ietf.org/html/rfc7671#section-7) and [RFC 5321](https://tools.ietf.org/html/rfc5321#section-5.1) imply that the usage of a CNAME in MX records is allowed.
* Using Server Name Indication (SNI) in an e-mail environment (for matching the certificate offered by a recieving e-mail server) is only usefull when DANE and DNSSEC are used. DNSSEC to perform a reliable MX lookup and DANE to verify the authenticity of the certificate. Sending e-mail servers (the TLS client) usually don't use SNI, because some receiving e-mail servers (the TLS server) cannot handle this; in some cases the setting up of a TLS connection fails. For more information see [RFC 7672 section 8.1](https://tools.ietf.org/html/rfc7672#section-8.1) and [this blogpost by Filippo Valsorda](https://blog.filippo.io/the-sad-state-of-smtp-encryption/). 
* Make sure you keep an eye on the logs of your sending mail server to see what domains fail DANE verification.
* Some software allows for a test mode. This means that DANE verification is done and logged but there’s no consequence for delivery if DANE verification fails.
* Manually verify a mail server's certificate with the following commands:
  * get the DANE record: `dig tlsa _25._tcp.mail.example.nl`
  * verify certificate against TLSA record `openssl s_client -starttls smtp -connect mail.example.nl:25 -dane_tlsa_domain "mail.example.nl" -dane_tlsa_rrdata "3 1 1 29c8601cb562d00aa7190003b5c17e61a93dcbed3f61fd2f86bd35fbb461d084"`.
* Check if DANE TLSA records (_25._tcp.mail.example.nl) are properly DNSSEC signed. A regularly occuring mistake is the presence of "proof of non-existence" (NSEC3) for the ancestor domain (_tcp.mail.example.nl). If this happens then resolvers that use qname minimization (like the resolver used by [Internet.nl](https://internet.nl)) think that _25._tcp.mail.example.nl does not exists since _tcp.mail.example.nl does not exists. Therefore the resolver can't get the TLSA record which makes DANE fail. 
  * Check your DNSSEC implementation on [DNSViz](https://dnsviz.net/). Enter "_25._tcp.mail.example.nl".
  * You can also manually check for this error. `dig _25._tcp.mail.example.nl tlsa +dnssec` results in a NOERROR response, while `dig _tcp.mail.example.nl tlsa +dnssec` results in a NXDOMAIN response. 

## DANE roll-over records
1. issue the new certificate (do not install)
2. generate and add the new dns-record (do not remove the previous record)
3. wait at least double of the records time to live (TTL). When you record is 5min. then you would wait 11min. or longer.
4. now install the new certificate
5. restart your mail server and verify that it is using the new certificate
6. now you can remove the old record

# Implementations

These specific write-ups show you how to use DANE SMTP for incoming and/or outgoing email traffic:

* [Postfix](DANE-for-SMTP-how-to-Postfix.md)
* [Exim](DANE-for-SMTP-how-to-Exim.md)
* [Halon](DANE-for-SMTP-how-to-Halon.md)
* [PowerMTA on port 25 (outbound only)](DANE-for-SMTP-how-to-PowerMTA.md)

# Special thanks
Our infinite gratitude goes out to the following people for their support in building this how-to for DANE.

Alexander Brotman  
Alwin de Bruin  
Anders Berggren  
Marc van de Geijn  
Mark Scholten  
Patrick Koetter  
Simon Besteman  
Tom van Leeuwen  
Viktor Dukhovni  
Wido Potters  
