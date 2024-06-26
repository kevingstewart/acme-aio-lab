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
A testing environment is contained within this repository in the form of a Docker Compose file. This ["all-in-one" Docker Compose](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-docker-compose.yaml) creates the following services needed to build a fully self-contained, container-based, client-server ACMEv2 testing lab:

- DNS server (bind9)
- (2) ACME server implementations
  - Pebble
  - SmallStep
- (2) ACME client implementations
  - NGINX (configured as a simple TLS web server, with [Certbot](https://eff-certbot.readthedocs.io/en/stable/intro.html) ACME client)
  - Netshoot (generic network utility container, with Certbot ACME client)

The compose file builds two networks - one internal managed by the DNS server, and one external for some services to expose ports outside the environment (specifically NGINX).

![acme-aio-lab-basic-lab-diagram](https://github.com/kevingstewart/acme-aio-lab/assets/16813250/d54a3083-b039-4298-ae00-dbc66ddd156e)


