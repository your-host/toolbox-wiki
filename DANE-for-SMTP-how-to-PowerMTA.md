# Implementing DANE for SMTP on Port25 PowerMTA (outbound e-mail traffic)
Various Email Service Providers (ESP) are using Port25 PowerMTA as their preferred outbound MTA.

Port25 has build-in support for DANE since version v5.0

**Specifics for this setup**
* Port25 PowerMTA v5.x+

**Assumptions**
* Basic configuration of PowerMTA
* TLS Certificate already in place
* STARTTLS active (`use-starttls    yes`)
* DNSSEC capable DNS resolver

## Generic configuration
### Configure the use of DANE on the domain level

Within the configuration of a specific domain
```
  <domain example.com>
    use-dane true
  </domain>
```

  Within the configuration of all domains
```
    <domain *>
      use-dane true
    </domain>
```

If TLS is required per DANE for SMTP `use-starttls` and `require-starttls` will be overridden.

### Logging

By enabling the resolver logging (`log-resolution yes`) you are able to see the DNS queries for the TLSA records.

Within the configuration of a specific domain
```
  <domain example.com>
    log-resolution yes
  </domain>
```

  Within the configuration of all domains
```
    <domain *>
      log-resolution yes
    </domain>
```

By enabling the TLS logging (`log-tls yes`) you are able to see the TLS DANE verification.

Within the configuration of a specific domain
```
  <domain example.com>
    log-tls yes
  </domain>
```

  Within the configuration of all domains
```
    <domain *>
      log-tls yes
    </domain>
```

Example logging:

DNS Resolver
```
Starting new query for TLSA _25._tcp.jail.internet.nl:
answer _25._tcp.jail.internet.nl. 3600 IN CNAME 3.1.1._dane.internet.nl.
answer _25._tcp.jail.internet.nl. 3600 IN CNAME 3.1.1._dane.internet.nl.
answer 3.1.1._dane.internet.nl. 3600 IN TLSA 3 1 1 25DE2127E359B8522DDD6E237381458804549CDE5440E4F5B547C5629E48D46B
_25._tcp.jail.internet.nl. 3600 IN CNAME 3.1.1._dane.internet.nl.
_25._tcp.jail.internet.nl. 3600 IN CNAME 3.1.1._dane.internet.nl.
3.1.1._dane.internet.nl. 3600 IN TLSA 3 1 1 25DE2127E359B8522DDD6E237381458804549CDE5440E4F5B547C5629E48D46B
```
TLS Logging
```
tls: TLSv1.2 connected with 256-bit ECDHE-RSA-AES256-GCM-SHA384
tls: cert: /C=NL/ST=Zuid-Holland/L=Den Haag/XXXXXXX; issuer=/C=NL/O=XXXXXXX; verified=yes
tls: DANE match: /C=NL/ST=Zuid-Holland/L=XXXXXX; issuer=XXXXX; depth=0
```
