### ACMEv2 All-in-One Testing Lab for DNS-01 Proof Validation
All ACME proof validations must prove ownership of *something*. For the **dns-01** challenge, the client must prove:

- Ownership of the DNS domain
- Ownership and control over DNS records in that domain

<br />

That is, the client must simply be able to control the DNS records for the requested domain. The following description of the protocol exchange is super-simplistic for the sake of illustrating the dns-01 proof requirement, and assumes things like client registration are complete. A more detailed protocol exchange is included further down in this document. Also for the sake of this document, the term "ACME provider", or simply "provider" is used to indicate an ACME service (ex. Let's Encrypt).

#### References
ACMEv2 dns-01 is covered in the following references:
- https://datatracker.ietf.org/doc/html/rfc8555#section-8.4

<br />


#### Simplified ACME DNS-01 Protocol Exchange
- An ACME client sends a request to an ACME provider to "order" a new certificate (ex. www.example.com).
- The ACME provider sends back a list of supported proof validation options, typically **http-01**, **dns-01**, and **tls-alpn-01**, and the client indicates that it wants to use **dns-01**.
- The provider then sends a validation token (ex. _4FirTtbveHPsRiDRxAVOgvYsaCSOzPGy_)
- The client must now create a DNS TXT record, named "_acme-challenge.\<domain\>" and add the validation token as the TXT record value. The client then tells the provider that it's ready. Ex.
  ```
  _acme-challenge.www.example.com    120  IN    TXT    4FirTtbveHPsRiDRxAVOgvYsaCSOzPGy
  ```
- The provider may take some time to perform its validation, so the client will periodically poll the provider for status. Once the provider is done and the status is good, the client will send a Certificate Signing Request (CSR) for the requested certificate.
- The provider will create the certificate and send the client a URL it can use to fetch it.
- The client fetches the new certificate from this URL and should now remove the DNS TXT record.

This is the general flow of ACMEv2 dns-01. Again, a more detailed and correct description is added below for reference. Once the client has access to the new certificate it may need to push that to the TLS service that actually needs it. The provisions for this step are specific to the TLS service. Also, in a real world scenario, the DNS is some Internet-based entity (ex. Cloudflare, Google), so the client has no control over this service other than to write records for its own zone. This is the essential premise of the dns-01 ACME proof validation.

<br />

-----

### Testing ACMEv2 DNS-01 in the All-in-One Lab Environment
The ACMEv2 all-in-one lab consists of a Docker Compose file that builds all of the necessary components to support a fully-contained, container-based environment. No external services are required. For DNS-01 testing specifically, you need an ACME client (NGINX server with acme.sh client), an ACME provider (Pebble or Smallstep), and a DNS server (Bind). During the proof phase of the protocol exchange, the client must be able to create a DNS TXT record, and subsequently remove it. In the lab, the ACME client is built with a set of scripts to work at the different phases:

<details>
  <summary>A DNS RFC2136 DNS Update Configuration</summary>
  The acme.sh client will utilize an RFC2136 DNS Update configuration to enable the client to make dynamic updates to the DNS records on the Bind DNS server. This involves the establishment of an encryption key and update-policy applied to the specific Bind DNS zone, and the encryption key shared to the client.

  <br />
  
</details>
<details>
  <summary>A "deploy hook" script moves the certificate into place and reloads the TLS server</summary>
  The simple TLS configuration on the NGINX server defines a location for the certificate. This script moves the renewed certificate to that location and reloads NGINX. This script is called from the acme.sh ```--deploy``` function.

  ```shell
  #!/usr/bin/bash
  nginx_local_deploy() {
      _cdomain="$$1"
      _ckey="$$2"
      _ccert="$$3"
      _cca="$$4"
      _cfullchain="$$5"

      cp -f "$$_ccert" /etc/letsencrypt/live/f5labs.local/cert.pem
      cp -f "$$_ckey" /etc/letsencrypt/live/f5labs.local/privkey.pem
      cp -f "$$_cfullchain" /etc/letsencrypt/live/f5labs.local/chain.pem
      nginx -s reload
  }
  ```
</details>

<br />

**To test dns-01**:

1. Start the Docker Compose environment.
   ```shell
   docker compose -f acme-aio-internal-compose.yaml up -d
   ```
2. Tail the NGINX container log until the logs settles. Many things are happening under the hood.
   ```shell
   docker logs -f nginx
   ```
3. From your local system make an HTTPS request to the NGINX container, port 8443. On initial start this will use a "default" certificate.
   ```shell
   curl -vk https://(docker-host-ip):8443
   ```
4. Shell into the NGINX container.
   ```shell
   docker exec -it nginx /bin/bash
   ```
5. From the NGINX container, execute an ACME certificate renewal request to one of the ACME providers for a new certificate. The -vvv option on the command line will dump the entire ACME protocol exchange for your review. The below example uses the Pebble ACME server for the www.f5labs.local domain and specifies each of the hook scripts. The first two "NSUPDATE" variables are used by the acme.sh client to perform an RFC2136 remote DNS update to the Bind server. By default, acme.sh will attempt to use public DNS to check that the DNS entry has been created. This doesn't work in a self-contained lab environment, so the "--dnssleep 20" option is added here to instruct acme.sh to pause (20 seconds) and not attempt the external DNS check.
   ```shell
   export NSUPDATE_KEY=/root/rfc2136-acmesh.ini
   export NSUPDATE_SERVER=10.10.0.53
   export SERVER=https://pebble.acmelabs.local:14000/dir
   #export SERVER=https://smallstep.acmelabs.local:9000/acme/acme/directory
   export DOMAIN=www.f5labs.local
   /root/acme/acme.sh --issue --dns dns_nsupdate --dnssleep 20 --insecure --server ${SERVER} -d ${DOMAIN} --force && \
   /root/acme/acme.sh --deploy -d ${DOMAIN} --deploy-hook nginx_local
   ```
6. From the NGINX container, view the properties of the renewed certificate.
   ```shell
   openssl x509 -noout -text -in /etc/letsencrypt/live/f5labs.local/cert.pem
   ```
7. From your local system make another HTTPS request to the NGINX container, port 8443. This should now be using the renewed certificate.
   ```shell
   curl -vk https://(docker-host-ip):8443
   ```
8. Optionally, shut down the Docker Compose when you are done testing. This will reset all configuration data.
   ```shell
   docker compose down
   ```

<br />

-----
### Detailed ACMEv2 DNS-01 Protocol Exchange

The ```--debug``` option in the acme.sh command will print out all of the protocol exchange messages. The following section elaborates on each of these messages.

<details>
  <summary>1. Directory service request</summary>
  <br />
  This is the only URL that is required to be known in advance, as the response will list the URLs for the other services. Within the directory listing there should minimally be resources for "NewAccount" (registration), "newNonce" (getting a new nonce), and "newOrder" (requesting certificate(s)). Optionally there may also be "revokeCert" (revoke an issued certificate) and "keyChange" (rotate registration key) services.
  <br />
  
  ```
  GET https://pebble.acmelabs.local:14000/dir
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  {
     "keyChange": "https://pebble.acmelabs.local:14000/rollover-account-key",
     "meta": {
        "externalAccountRequired": false,
        "termsOfService": "data:text/plain,Do%20what%20thou%20wilt"
     },
     "newAccount": "https://pebble.acmelabs.local:14000/sign-me-up",
     "newNonce": "https://pebble.acmelabs.local:14000/nonce-plz",
     "newOrder": "https://pebble.acmelabs.local:14000/order-plz",
     "revokeCert": "https://pebble.acmelabs.local:14000/revoke-cert"
  }
  ```
</details>
<details>
  <summary>2. New nonce request (newNonce service)</summary>
  <br />
  All subsequent requests must contain a Nonce value to protect against replay attacks. To get the initial nonce the client makes a HEAD request to the "newNonce" service URL, which is then returned in a "Replay-Nonce" header.
  <br />
  
  ```
  HEAD https://pebble.acmelabs.local:14000/nonce-plz
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: by-pX5V5rET91YIwd0qzJw
  ```
</details>
<details>
  <summary>3. Registration request (newAccount service)</summary>
  <br />
  Assuming the client has not yet registered with the ACME provider, it needs to first make a POST request to the "newAccount" service. The content of the request payload includes a "payload" block containing the "contact" email address and agreement to the provider's terms-of-service, a "protected" block that contains the previous nonce, service URL, and JSON web key attributes (algorithm, key type, modulus[n], and exponent[e]), and a "signature" block that is a digital signature using the client's private key. Note that in this and all following requests, the "protected" and "payload" blocks are base64-encoded. These are shown decoded here to better understand the protocol exchange. Also note that the provider should return a new nonce value in each response, which the client should use in the subsequent request.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/sign-me-up
  {
    "protected": {
        "alg": "RS256", 
        "jwk": {
           "n": "yNZZe54dnQk_KggAbe-txbibe-...", 
           "e": "AQAB", 
           "kty": "RSA"
        }, 
        "nonce": "by-pX5V5rET91YIwd0qzJw", 
        "url": "https://pebble.acmelabs.local:14000/sign-me-up"
     },
    "signature": "...",
    "payload": {
        "contact": [
           "mailto:admin@f5labs.local"
        ],
        "termsOfServiceAgreed": true
     }
  }
  -------------------------------------------
  HTTP 201
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Location: https://pebble.acmelabs.local:14000/my-account/1
  Replay-Nonce: VHJYFJtDzXaxnu2Ohm4O7w
  {
     "status": "valid",
     "contact": [
        "mailto:admin@f5labs.local"
     ],
     "orders": "https://pebble.acmelabs.local:14000/list-orderz/1",
     "key": {
        "kty": "RSA",
        "n": "yNZZe54dnQk_KggAbe-txbibe-...",
        "e": "AQAB"
     }
  }
  ```
</details>
<details>
  <summary>4. Certificate request (newOrder service)</summary>
  <br />
  The client is now request to request a new certificate. To do that it makes a POST request to the "newOrder" service URL, and in that request it supplies a similar (base64-encoded) "protected" block, a (base64-encoded) "payload" block that contains an "identifiers" array of domain names (the certificate domains requested), and "signature" block. The provider will return two important URLs:
  <br />
  
  - authorizations: an array listing the URL(s) to query to get challenge information
  - finalize: the URL that will be used once the challenges are successful
  
  ```
  POST https://pebble.acmelabs.local:14000/order-plz
  {
  "protected": {
      "alg": "RS256", 
      "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
      "nonce": "VHJYFJtDzXaxnu2Ohm4O7w", 
      "url": "https://pebble.acmelabs.local:14000/order-plz"
   },
  "signature": "...",
  "payload": {
      "identifiers": [
         {
            "type": "dns",
            "value": "www.f5labs.local"
         }
      ]
   }
}
-------------------------------------------
HTTP 201
Cache-Control: public, max-age=0, no-cache
Content-Type: application/json; charset=utf-8
Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
Location: https://pebble.acmelabs.local:14000/my-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs
Replay-Nonce: cKc9heXQdLmojUINiJOMoA
{
   "status": "pending",
   "expires": "2024-06-28T21:07:14Z",
   "identifiers": [
      {
         "type": "dns",
         "value": "www.f5labs.local"
      }
   ],
   "finalize": "https://pebble.acmelabs.local:14000/finalize-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs",
   "authorizations": [
      "https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ"
   ]
}
  ```
</details>
<details>
  <summary>5. Authorizations Request</summary>
  <br />
  The client sends its request with "protected" block, an empty "payload" block, and the "signature" block. The authorizations request should return an array of "challenges" - the set of proof validation functions (ex. http-01, dns-01, tls-alpn-01) and corresponding ephemeral validation tokens. 
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "cKc9heXQdLmojUINiJOMoA", 
        "url": "https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ"
     },
    "signature": "...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: 3kVnFRJYPyLZnLEAilf8AA
  {
     "status": "pending",
     "identifier": {
        "type": "dns",
        "value": "www.f5labs.local"
     },
     "challenges": [
        {
           "type": "http-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/cHHW1Ao2mu_ckCcwB6cSFlLxdpPMl4ZW2KGgfvroBRc",
           "token": "4GDRx8S77JRXFFM4KikAEWeSc1R5AaELV4OzXWxap24",
           "status": "pending"
        },
        {
           "type": "dns-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/VQM9vxUsiakiKOo6R1wQg4_zS9-UJqMAnf4MPGiuNDU",
           "token": "iBNF15sfcOKMa0i1SNVVJFGBya85VFLLxO15X1aXFKg",
           "status": "pending"
        },
        {
           "type": "tls-alpn-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/yhS1mUTHinVjQsb_rXVlj1aDLXrfCm5r0bnRfApIT9U",
           "token": "Uuoyh7pIMyEEEO-KBFLdcDmeZrsdjbJhJ8DA0HIJOLM",
           "status": "pending"
        }
     ],
     "expires": "2024-06-27T22:07:14Z"
  }
  ```
</details>
<details>
  <summary>6. Client Function: Stage the DNS TXT record</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target DNS resource. For public DNS like Cloudflare, this is usually handled with an API and API key(s). The goal is to insert a DNS TXT record for this domain (zone). Proof validation is established by virtue of the fact that the client only owns/manages DNS records for this resource in a public DNS service. For the sake of completeness, however, the acme.sh client uses an RFC2136 DNS Update configuration and shared encryption key to remotely update the DNS zone. In this specific instance, the validation value is "iBNF15sfcOKMa0i1SNVVJFGBya85VFLLxO15X1aXFKg", the dns-01 token value from the authorizations response.
  <br />
</details>
<details>
  <summary>7. Let the provider know the challenge is ready</summary>
  <br />
  Notice also the "url" value in the dns-01 block of the authorizations response. This URL is how the client will indicate its preference to use dns-01 proof validation. The client needs to make a POST request to this URL, pass in "protected" block, empty "payload" block, and the "signature" block. The provider will return the same dns-01 authorizations block with a "pending" status, indicating it will commence validation.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/chalZ/VQM9vxUsiakiKOo6R1wQg4_zS9-UJqMAnf4MPGiuNDU
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "3kVnFRJYPyLZnLEAilf8AA", 
        "url": "https://pebble.acmelabs.local:14000/chalZ/VQM9vxUsiakiKOo6R1wQg4_zS9-UJqMAnf4MPGiuNDU"
     },
    "signature": "...",
    "payload": "{}"
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index", <https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ>;rel="up"
  Replay-Nonce: ve5MPLzO1b1JrZ_xtH7Y_g
  {
     "type": "dns-01",
     "url": "https://pebble.acmelabs.local:14000/chalZ/VQM9vxUsiakiKOo6R1wQg4_zS9-UJqMAnf4MPGiuNDU",
     "token": "iBNF15sfcOKMa0i1SNVVJFGBya85VFLLxO15X1aXFKg",
     "status": "pending"
  }
  ```
</details>
<details>
  <summary>8. Poll the provider for validation status</summary>
  <br />
  A busy ACME provider may take some time to get to this validation, so the client should continue to poll the provider for status. To do that it makes a POST request to the same authorizations URL, passing in "protected" block, empty "payload" block, and the "signature" block. Once the provider has had a chance to validate the challenge (query the DNS TXT record) it will return a response to the client's poll indicating a "valid" status.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "ve5MPLzO1b1JrZ_xtH7Y_g", 
        "url": "https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ"
     },
    "signature": "...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: pTzDsi6NbEs00NaH54jCSQ
  {
     "status": "valid",
     "identifier": {
        "type": "dns",
        "value": "www.f5labs.local"
     },
     "challenges": [
        {
           "type": "dns-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/VQM9vxUsiakiKOo6R1wQg4_zS9-UJqMAnf4MPGiuNDU",
           "token": "iBNF15sfcOKMa0i1SNVVJFGBya85VFLLxO15X1aXFKg",
           "status": "valid",
           "validated": "2024-06-27T21:07:20Z"
        }
     ],
     "expires": "2024-06-27T22:07:20Z"
  }
  ```
</details>
<details>
  <summary>9. Client Function: Clean up the DNS TXT record</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target DNS resource. For public DNS like Cloudflare, this is usually handled with an API and API key(s). The goal is simply to remove the previous DNS TXT record for this domain (zone). 
  <br />
</details>
<details>
  <summary>10. Send a Certificate Signing Request</summary>
  <br />
  As previously noted, the "finalize" URL that came from the newOrder request is to be used once the proof validation is successful. The client needs to make a POST request this URL, sending the "protected" block, a "payload" block containing the certificate signing request (CSR), and the "signature" block. At this point that provider may return one of two things:
  <br />

  - A status of "processing" in which case the client needs to "poll" the order URL in the response "Location" header
  - A status of "valid" in which case it also provides a URL to fetch the new certificate

In the below we show the former "pending" state.
  
  ```
  POST https://pebble.acmelabs.local:14000/finalize-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "pTzDsi6NbEs00NaH54jCSQ", 
        "url": "https://pebble.acmelabs.local:14000/finalize-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs"
     },
    "signature": "...",
    "payload": {
        "csr": "MIHpMIGQAgEA..."
     }
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Location: https://pebble.acmelabs.local:14000/my-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs
  Replay-Nonce: Kudlh1GjiYtcD5GhUw2C9Q
  {
     "status": "processing",
     "expires": "2024-06-28T21:07:14Z",
     "identifiers": [
        {
           "type": "dns",
           "value": "www.f5labs.local"
        }
     ],
     "finalize": "https://pebble.acmelabs.local:14000/finalize-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs",
     "authorizations": [
        "https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ"
     ]
  }
  ```
</details>
<details>
  <summary>11. Polling Order Status</summary>
  <br />
  Assuming the status value is "processing" from the finalize-order request and no certificate URL has been returned, the client will continue to poll the for the order status, eventually getting back a status of "valid" and a certificate URL:
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/my-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "Kudlh1GjiYtcD5GhUw2C9Q", 
        "url": "https://pebble.acmelabs.local:14000/my-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs"
     },
    "signature": "...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: RTV4jEVkRYzvW3hTKvF9gg
  {
     "status": "valid",
     "expires": "2024-06-28T21:07:14Z",
     "identifiers": [
        {
           "type": "dns",
           "value": "www.f5labs.local"
        }
     ],
     "finalize": "https://pebble.acmelabs.local:14000/finalize-order/g18GvKI-u7f4XaM8GsawoZbx0D1wZrNqNO0zBgnbAfs",
     "authorizations": [
        "https://pebble.acmelabs.local:14000/authZ/ttC1OkA8mAP9KgXMVjSK3CgdIGv-NWTuIQpw5P2AWYQ"
     ],
     "certificate": "https://pebble.acmelabs.local:14000/certZ/14866f1c6cce8a10"
  }
  ```
</details>
<details>
  <summary>12. Retrieve Certificates</summary>
  <br />
  Once the provider returns the certificate URL, it can use this URL to fetch the new certificate. The provider will usually send both the renewed certificate and its issuer. The certificate(s) will be in PEM format.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/certZ/14866f1c6cce8a10
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "RTV4jEVkRYzvW3hTKvF9gg", 
        "url": "https://pebble.acmelabs.local:14000/certZ/14866f1c6cce8a10"
     },
    "signature": "...",
    "payload": ""
  }
  
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/pem-certificate-chain; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index", <https://pebble.acmelabs.local:14000/certZ/14866f1c6cce8a10/alternate/1>;rel="alternate"
  Replay-Nonce: ITfrKlU1dwmpxr1LsYBShA
  Transfer-Encoding: chunked
  
  -----BEGIN CERTIFICATE-----
  MIICmDCCAYCgAwIBAgIIFIZvHGzOihAwDQYJKoZIhvcNAQELBQAwKDEmMCQGA1UE
  ...
  sPeTXGqMvazUTjs51UMjTkRFtFUJlGh8HoO86iFJbl5pJsma4OL69aeHtTk=
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  MIIDUDCCAjigAwIBAgIIXn5x8Zi3Ds0wDQYJKoZIhvcNAQELBQAwIDEeMBwGA1UE
  ...
  nQn5+/5xCqTFELxCKRm8pJ9KmGC1lfahS6se+TUSU5FUn3CO
  -----END CERTIFICATE-----
  ```
</details>
<details>
  <summary>13. Client Function: Move the Certificate(s) into Position</summary>
  <br />
  Wherever the ACME client may be running, it now needs to move the new certificate(s) into position where the TLS server needs them. In the case of a server like NGINX, it also needs to reload the configuration data to update the certificates in memory. For the sake of completeness, this lab's Bash script is included that simply copies the renewed certificates into the location that NGINX is expecting, and then issues a config reload. This process is otherwise highly dependent on the TLS server. The following script is called from the acme.sh ```--deploy``` function.
  <br />
  
  ```
  #!/usr/bin/bash
  nginx_local_deploy() {
      _cdomain="$$1"
      _ckey="$$2"
      _ccert="$$3"
      _cca="$$4"
      _cfullchain="$$5"

      cp -f "$$_ccert" /etc/letsencrypt/live/f5labs.local/cert.pem
      cp -f "$$_ckey" /etc/letsencrypt/live/f5labs.local/privkey.pem
      cp -f "$$_cfullchain" /etc/letsencrypt/live/f5labs.local/chain.pem
      nginx -s reload
  }
  ```
</details>

While there are variations not discussed here (ex. key rotation), the above summarizes a generic ACME protocol exchange for dns-01 proof validation.


<br />

-----
### Additional Lab Environment Notes

- The Smallstep ACME provider will sign new certificates using the embedded root CA certificate and key, but Pebble will create a new CA root and intermediate certificates and keys on each startup. To get to the Pebble CA certificates, issue the following Curl commands from within either of the ACME client hosts (NGINX or Netshoot)

  ```
  curl -sk https://pebble.acmelabs.local:15000/roots/0
  curl -sk https://pebble.acmelabs.local:15000/intermediates/0
  ```




















