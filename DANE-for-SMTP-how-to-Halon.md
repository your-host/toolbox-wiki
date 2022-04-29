- [Implementing DANE for SMTP on Halon (inbound & outbound e-mail traffic)](#implementing-dane-for-smtp-on-halon--inbound---outbound-e-mail-traffic-)
  * [Generic configuration](#generic-configuration)
    + [Install or generate key pair](#install-or-generate-key-pair-1)
    + [DNSSEC](#dnssec)
  * [Inbound](#inbound)
  * [Outbound](#outbound)
    + [Transport Label](#transport-label)
    + [Scripting](#scripting)
    + [Logging](#logging)

# Implementing DANE for SMTP on Halon (inbound & outbound e-mail traffic)
Several Dutch hosting providers use Halon (a scriptable SMTP server who's virtual appliances are based on FreeBSD) as the internet facing e-mail server. The actual mail boxes reside on Direct Admin (which uses Exim) within the internal network. In this specific setup you could say that all security features are applied at the internet facing mail server which is Halon. 

Halon has built-in support for DANE and can be configured in several ways, this how-to uses the built-in web interface. For more information on configuration you can visit [https://halon.io/dane](https://halon.io/dane) and [https://wiki.halon.io/DANE](https://wiki.halon.io/DANE).

**Specifics for this setup**
* HALON 5.1-p3-argy

**Assumptions**
* Basic configuration of Halon

## Generic configuration
### Install or generate key pair

Navigate to: `Configuration -> Email engine - > Certificates and keys -> Add`

- Give an ID (name) for the certificate: only lowercase, numbers and letters (limitation of Halon). 
- Select type “X.509 and private key” and hit *Generate* and fill in the fields or upload a certificate (chain with private key).
- Add a comment like name of the CN and date of validity.

### DNSSEC
If you have a local (in your own network) DNSSEC resolver available, you can use that one. Otherwise use the internal resolver of Halon, which is based on unbound.

Navigate to: `Hosts -> Network -> DNS`
Name servers: enter here your resolving nameservers.
DNS cache: Enable (now the DNSSEC option is visible)
DNSSEC: Enable

![](images/dane-halon-dnssec.png)

If you have multiple hosts in a cluster, edit the DNS settings for all the hosts.

## Inbound
Navigate to: `Configuration -> Email engine -> Settings`

If you already have an inbound SMTP listener configured, you should upgrade this one to DANE. You can also create a new listener by navigating to `Configuration -> Server -> SMTP listeners -> Add`

- STARTTLS: Enable
- Certificate: Select the certificate you want to use from the dropdown menu. 

![](images/dane-halon-inbound.png)

## Outbound
There a multiple ways to enable outbound DANE: through a Transport Label or scripting.

### Transport Label
Navigate to: `Configuration -> Email Engine -> Transport labels`

Select the transport labels ID if you want to upgrade an existing one, otherwise click *Add* to create a new one.

- Destination hostname and port: Select Deliver to MX.
- TLS STARTTLS: Select "DANE" or "DANE (required)" in the dropdown menu.

![](images/dane-halon-outbound.png)

Notice: with "DANE (required)" there is no fallback to TLS or none option.

If you created a new "Transport Label" use that one in a Script mapping as Transport to use the DANE enabled delivery.

### Scripting
With the SetTLS function in pre-delivery you can set the TLS options for that specific delivery attempt.

`SetTLS(["tls" => "dane”]);`

`SetTLS(["tls" => "dane_require”]);`

### Logging

In the logging you can see whether delivery via DANE is used.

Example logging:
```
Processing message from <mail@example.nl> to <danedomain@example.nl> with daneenabled
Delivering message to example.nl:25
Connecting to [192.168.1.1]:25 (DNSSEC)
Connection is now using TLSv1.3 with cipher TLS_AES_256_GCM_SHA384 (256 bits) (DANE)
Successful delivery to <danedomain@example.nl> (retry 0) in 2.108s: 2.0.0 example.org accepted message 0123456789ABCD
Deleting mail file (all recipients done)
```
