# ACMEv2 All-in-One Testing Lab
An ACMEv2 all-in-one testing lab that supports http-01, dns-01, and tls-alpn-01 challenges

### Introduction
ACME is an *automated* certificate renewal protocol that relies on "proof of ownership of **something**". While it is intended to be extensible, the standard ACMEv2 (version 2) implementation generally supports three forms of proof validation:

* DNS (dns-01) - proof of ownership of a domain
* HTTP (http-01) - proof of ownership of a domain and application asset
* TLS (tls-alpn-01) - proof of ownership of a domain ...

The above linked pages describe each proof validation in more detail, as a series of client-server challenge and response functions. And within these pages, specific testing scenarios are also covered.

----

### Testing Environment
A testing environment is contained within this repository in the form of a Docker Compose file. This "all-in-one" Docker Compose creates the following services needed to define a fully self-contained client-server ACMEv2 testing lab:

- DNS server (bind9)
- (2) ACME server implementations
  - Pebble
  - SmallStep
- (2) ACME client implementations
  - NGINX (configured as a simple TLS web server)
  - Netshoot (generic network utility container)



