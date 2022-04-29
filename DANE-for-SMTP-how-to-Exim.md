# Implementing DANE for SMTP on Exim (inbound & outbound e-mail traffic)
**Specifics for this setup**
* Ubuntu 18.10 ‘Cosmic Cuttlefish’ 
* Exim 4.92 (DANE support is non-experimental since version 4.91)

**Assumptions**
* DNSSEC is used
* Mail server is operational

## Configuration for inbound e-mail traffic

### Install or generate key pair
You can use a commercial or Let's Encrypt certificate, but you can also generate your own key pair by using the provided Exim tools. Use the following command to generate a key pair.

`sudo bash /usr/share/doc/exim4-base/examples/exim-gencert`

### Configure TLS 
In Exim you should configure TLS by adding the following to **main/03_exim4-config_tlsoptions**

    MAIN_TLS_ENABLE = yes
    tls_advertise_hosts = *
    tls_certificate = /path/to/certificate.crt
    tls_privatekey = /path/to/private.key

## Configuration for outbound e-mail traffic
This part of the how-to describes the steps that should be taken with regard to your outbound e-mail traffic. This enables your e-mail environment to use DANE for validating the certificates offered by other e-mail servers. 

### DNSSEC validating resolvers
Make sure to configure DNSSEC validating resolvers on the mail server. When using the locale systemd resolver, make sure to add the following to **/etc/systemd/resolved.conf**.

`DNSSEC = yes`

### Configure DNSSEC validation in Exim
In Exim you explicitly need to configure DNSSEC validation by adding the following to **main/02_exim4-config_options** since some resolvers only validate DNSSEC on request. 

`dns_dnssec_ok = 1`

### Configure DANE
In order to use DANE, you should tell Exim to check for DANE records when sending e-mail. You can configure DANE validation to be mandatory by adding the following to **transport/30_exim4-config_remote_smtp**. 

`hosts_require_dane = *`

This means that TLS connections are not accepted when the domain you are trying to send mail to does not have a valid TLSA record. Since this is rather strict and not recommended to be the default, you are probably better of by configuring DANE validation to be additional. This can be done by adding the following to **transport/30_exim4-config_remote_smtp**.

`hosts_try_dane = *`

Notice that depending on the way you configured Exim, you need to apply DANE for all [SMTP transports](https://www.exim.org/exim-html-current/doc/html/spec_html/ch-how_exim_receives_and_delivers_mail.html#SECTprocaddress).
