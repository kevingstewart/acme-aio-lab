# ACMEv2 All-in-One Testing Lab
An ACMEv2 all-in-one testing lab that supports http-01, dns-01, and tls-alpn-01 challenges

### Introduction
ACME is an *automated* certificate renewal protocol that relies on "proof of ownership of **something**". While it is intended to be extensible, the standard ACMEv2 (version 2) implementation generally supports three forms of proof validation:

* [DNS (dns-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-dns-01.md) - proof of ownership of a domain
* [HTTP (http-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-http-01.md) - proof of ownership of a domain and application asset
* [TLS (tls-alpn-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-tls-alpn-01.md) - proof of ownership of a domain ...

The above linked pages describe each proof validation in more detail, as a series of client-server challenge and response functions. And within these pages, specific testing scenarios are also covered. The purpose of this repository is to demonstrate the inner workings of the ACMEv2 protocol, within a fully self-contained, container-based, client-server ACMEv2 testing lab. No external resources are required (ex. DNS, Let's Encrypt, etc.) to operate this lab.

----

### Testing Environment
A testing environment is contained within this repository in the form of a Docker Compose file. This ["all-in-one" Docker Compose](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-docker-compose.yaml) creates the following services needed to build an ACMEv2 testing lab:

- DNS server (bind9)
- (2) ACME server implementations
  - Pebble
  - SmallStep
- (2) ACME client implementations
  - NGINX (configured as a simple TLS web server, with [Certbot](https://eff-certbot.readthedocs.io/en/stable/intro.html) ACME client)
  - Netshoot (generic network utility container, with Certbot ACME client)

The compose file builds two networks - one "internal" managed by the DNS server, and one "external" for some services to expose ports outside the environment (specifically NGINX).

<img src="https://github.com/kevingstewart/acme-aio-lab/assets/16813250/5c3f5fe9-efff-4ad9-8ed3-22bc01917711" width="60%">

The environment is configured as such:
- The entire internal network sits on 10.10.0.0/16.
- The DNS server listens on 10.10.0.53.
- The Pebble ACME server listens on 10.10.20.100.
- The Smallstep ACME server listens on 10.10.20.101.
- The Netshoot utility container listens on 10.10.10.5 and utilizes the Certbot ACME client.
- The NGINX server listens on 10.10.10.10, has an interface into the container "bridge" network to expose its 443 TLS port (8443 on the outside), utilizes the Certbot ACME client, and has a super-simple HTTPS listener configuration that returns "Hello World!".
- The DNS server enables recursion, and maintains two zones and subsequent zone entries:
  - f5labs.local 
    ```
    utility      IN      A      10.10.10.5
    *            IN      A      10.10.10.10
    ```
  - acmelabs.local
    ```
    pebble       IN      A      10.10.20.100
    smallstep    IN      A      10.10.20.101
    ```
- The Pebble ACME server's directory is at (accessible internally): https://pebble.acmelabs.local:14000/dir.
- The Smallstep ACME server's directory is at (accessible internally): https://smallstep.acmelabs.local:9000/acme/acme/directory.












