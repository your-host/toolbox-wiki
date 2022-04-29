# Implementing DANE for SMTP on Postfix (inbound & outbound e-mail traffic)

**Specifics for this setup**
* Linux Debian 9.8 (Stretch) 
* SpamAssassin version 3.4.2 (running on Perl version 5.28.1)
* Postfix 3.4.5
* BIND 9.10.3-P4-Debian
* Two certificates (for two mail servers) from Comodo / Sectigo

**Assumptions**
* DNSSEC is used
* Mail server is operational
* Software packages are already installed

## Configuring Postfix
Postfix plays an important role in using DANE for validating the when available.

Make sure the following entries are present in **/etc/postfix/main.cf**

`smtp_dns_support_level = dnssec`  

This setting tells Postfix to perform DNS lookups using DNSSEC. This is an important prerequisite for DANE to be effective, since regular DNS lookups can be manipulated. Without DNSSEC support, Postfix cannot use DANE.

`smtp_tls_security_level = dane`  

By default Postfix uses opportunistic TLS (smtp_tls_security_level = may) which is susceptible to man in the middle attacks. You could tell Postfix to use mandatory TLS (smtp_tls_security_level = encrypt) but this breaks backwards compatibility with mail servers that don't support TLS (and only work with plaintext delivery). However, when Postfix is configured to use the "dane" security level (smtp_tls_security_level = dane) it becomes resistant to man in the middle attacks, since Postfix will connect to other mail servers using "mandatory TLS" when TLSA records are found. This means that: 
* If TLSA records are found but are unusable, Postfix will NOT fallback to opportunistic TLS (STARTTLS) or ultimately plaintext delivery. 
* If no TLSA records are found (and thus DANE is not supported), Postfix will fallback to opportunistic TLS (STARTTLS) or ultimality plaintext delivery.

To enforce DANE you can use "smtp_tls_security_level = dane-only" (aka mandatory DANE). With this setting a sending mail server makes sure there is no fallback to opportunistic TLS (STARTTLS) or plaintext when either the TLSA records are unusable or absent. It is possible to use mandatory DANE on a per domain basis. For example when you know a certain domain should support DANE and you want to be absolutely sure that a secure and validated delivery is used and want to prevent fallbacks to a less secure delivery method. 

`smtp_host_lookup = dns`  

This tells Postfix to perform lookups using DNS. Although this is default behavior it is important to make sure this is configured, since DANE won't be enabled if lookups are performed using a different mechanism.

`smtpd_tls_CAfile = /path/to/ca-bundle-file.crt`  

When applying a DANE roll-over scheme using an "issuer certificate" (an intermediate or root certificate), Postfix must be able to provide the certificates of the used issuer in the chain of trust. Hence this setting.
