# ACMEv2 All-in-One Testing Lab
An ACMEv2 all-in-one testing lab that supports http-01, dns-01, and tls-alpn-01 challenges

### Introduction
ACME is an *automated* certificate renewal protocol that relies on "proof of ownership of **something**". While it is intended to be extensible, the standard ACMEv2 (version 2) implementation generally supports three forms of proof validation:

* [DNS (dns-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-dns-01.md) - proof of ownership of a domain
* [HTTP (http-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-http-01.md) - proof of ownership of a domain and application asset
* [TLS (tls-alpn-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-tls-alpn-01.md) - proof of ownership of a domain and TLS application asset (pending)...

The above linked pages describe each proof validation in more detail, as a series of client-server challenge and response functions. And within these pages, specific testing scenarios are also covered. The purpose of this repository is to demonstrate the inner workings of the ACMEv2 protocol, within a fully self-contained, container-based, client-server ACMEv2 testing lab. No external resources are required (ex. DNS, Let's Encrypt, etc.) to operate this lab.

----

### Self-Contained Testing Environment
A testing environment is contained within this repository in the form of a Docker Compose file. This ["all-in-one" Docker Compose](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-internal-compose.yaml) creates the following services needed to build an ACMEv2 testing lab:

- DNS server (bind9)
- (2) ACME servers
  - Pebble
  - SmallStep
- (2) ACME clients
  - NGINX (configured as a simple TLS web server, with [acme.sh](https://github.com/acmesh-official/acme.sh) ACME client)
  - Netshoot (generic network utility container (named "utility"), with acme.sh ACME client)

The compose file builds two networks - one "internal" managed by the DNS server, and one "external" for some services to expose ports outside the environment (specifically NGINX).

<img src="https://github.com/kevingstewart/acme-aio-lab/assets/16813250/5c3f5fe9-efff-4ad9-8ed3-22bc01917711" width="60%">

The environment is configured as such:
- The entire internal network sits on a 10.10.0.0/16 subnet.
- The DNS server listens on 10.10.0.53.
- The Pebble ACME server listens on 10.10.20.100.
- The Smallstep ACME server listens on 10.10.20.101.
- The Netshoot utility container (named "utility") listens on 10.10.10.5 and utilizes the acme.sh ACME client.
- The NGINX server listens on 10.10.10.10, has an interface into the container "bridge" network to expose its 443 TLS port (8443 on the outside), utilizes the acme.sh ACME client, and has a super-simple HTTPS listener configuration that returns "Hello World!".
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


----

### Local Testing Environment
A separate "local" testing environment is contained within this repository in the form of a Docker Compose file. This ["all-in-one local" Docker Compose](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-local-compose.yaml) creates the following services needed to build an ACMEv2 testing lab for your local lab environment:

- DNS server (bind9)
- (2) ACME servers
  - Pebble
  - SmallStep

The difference between this and the self-contained lab is that ACME clients will be running somewhere else in the network. For this to work:

- The external ACME client must add the Docker host as a DNS listener so that it can resolve the ACME providers (**pebble.acmelabs.local** and **smallstep.acmelabs.local**). The Bind container exposes TCP and UDP port 53 listeners to the Docker host.
- The **f5labs.local** and **acmelabs.local** zones in the Docker compose file must be updated to match the local lab environment.
  - In the acmelabs.local zone, the pebble and smallstep records should be the Docker host IP address.
  - In the f5labs.local zone, update the wildcard record and/or create additional records that point to the IP addresses of the external ACME client(s).
 
In the examples provided in the Docker compose file, the Docker host is at 172.16.1.114, so that's the IP used for the pebble and smallstep records in the acmelabs.local zone. The client is another virtual machine at 172.16.1.116, running Ubuntu, NGINX, and any ACME client software (ex. acme.sh, Certbot, Lego, etc.). The client is configured to add the Docker host as a DNS resource so that it can resolve the ACME providers, and the ACME providers use the same DNS service to resolve the client(s). The ACME client should now be able to request certificates for the f5labs.local domain.








