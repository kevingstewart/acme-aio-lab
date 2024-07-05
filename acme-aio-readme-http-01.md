### ACMEv2 All-in-One Testing Lab for HTTP-01 Proof Validation
All ACME proof validations must prove ownership of *something*. For the **http-01** challenge, the client must prove:

- Ownership of the DNS domain
- Ownership and control of the application asset

<br />

That is, the client must own the DNS record for the requested domain, and must have application-level control of the TLS service that the DNS record point to. The following description of the protocol exchange is super-simplistic for the sake of illustrating the http-01 proof requirement, and assumes things like client registration are complete. A more detailed protocol exchange is included further down in this document. Also for the sake of this document, the term "ACME provider", or simply "provider" is used to indicate an ACME service (ex. Let's Encrypt).

#### References
ACMEv2 http-01 is covered in the following references:
- [https://datatracker.ietf.org/doc/html/rfc8555#section-8.3](https://datatracker.ietf.org/doc/html/rfc8555#section-8.3)

<br />

#### Simplified ACME HTTP-01 Protocol Exchange
- An ACME client sends a request to an ACME provider to "order" a new certificate (ex. www.example.com).
- The ACME provider sends back a list of supported proof validation options, typically **http-01**, **dns-01**, and **tls-alpn-01**, and the client indicates that it wants to use **http-01**.
- The provider then sends a validation token (ex. _V8HttT5ph9UNpcIM0sF28B7dfFVtdH7P_)
- The client must now create an HTTP (port 80) listener at the same IP address as the TLS service and at a specifically-defined URL (/.well-known/acme-challenge/\<token-value\>) and respond to any requests to that URL with the token value. The client then tells the provider that it's ready.
- The provider may take some time to perform its validation, so the client will periodically poll the provider for status. Once the provider is done and the status is good, the client will send a Certificate Signing Request (CSR) for the requested certificate.
- The provider will create the certificate and send the client a URL it can use to fetch it.
- The client fetches the new certificate from this URL and should now remove HTTP listener configuration.

This is the general flow of ACMEv2 http-01. Again, a more detailed and correct description is added below for reference. Once the client has access to the new certificate it may need to push that to the TLS service that actually needs it. The provisions for this step are specific to the TLS service. Also, in a real world scenario, the DNS is some Internet-based entity (ex. Cloudflare, Google), so the client may have no control over this service, and only have control over the actual target application asset. This is the essential premise of the http-01 ACME proof validation.

<br />

-----

### Testing ACMEv2 HTTP-01 in the All-in-One Lab Environment
The ACMEv2 all-in-one lab consists of a Docker Compose file that builds all of the necessary components to support a fully-contained, container-based environment. No external services are required. For HTTP-01 testing specifically, you need an ACME client (NGINX server), an ACME provider (Pebble or Smallstep), and a DNS server (Bind). During the proof phase of the protocol exchange, the client must be able to create an HTTP port 80 listener configuration, and subsequently remove it. 

Note that in this lab the Cerbot ACME client will use the **standalone** option, which will create the HTTP port 80 listener internally. As the Certbot agent is running in the same container as the NGINX TLS service, the DNS IP will be the same. Once the Certbot client has the new certificate, it just needs to push that to NGINX and reload the TLS service configuration. This is done with a simple "deploy hook" Bash script:

<details>
  <summary>HTTP-01 "deploy hook" script to move the certificate into place and reload the TLS server</summary>
  
  ```shell
  #!/bin/bash
  cp -f /etc/letsencrypt/live/${RENEWED_DOMAINS}/* /etc/letsencrypt/live/f5labs.local/
  nginx -s reload
  ```
</details>

<br />

**To test http-01**:

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
5. From the NGINX container, execute an ACME certificate renewal request to one of the ACME providers for a new certificate. The -vvv option on the command line will dump the entire ACME protocol exchange for your review. The below example uses the Pebble ACME server for the www.f5labs.local domain and specifies the deploy hook script.
   ```shell
   server=https://pebble.acmelabs.local:14000/dir
   domain=www.f5labs.local
   certbot certonly --standalone \
   --no-eff-email --email admin@f5labs.local \
   -vvv --no-verify-ssl --agree-tos \
   --server ${server} \
   --domains ${domain} \
   --deploy-hook "/acme-hook-deploy.sh"
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
### Detailed ACMEv2 HTTP-01 Protocol Exchange

The ```-vvv``` option in the Certbot command will print out all of the protocol exchange messages. The following section elaborates on each of these messages.

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
  Replay-Nonce: _1lC0k2yni8FlVMY0bnaKA
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
        "nonce": "_1lC0k2yni8FlVMY0bnaKA", 
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
  Replay-Nonce: _1lC0k2yni8FlVMY0bnaKA
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
        "nonce": "_1lC0k2yni8FlVMY0bnaKA", 
        "url": "https://pebble.acmelabs.local:14000/order-plz"
    },
    "signature": "knkitI-EQ0V4SgDCjlFTCraBjy...",
    "payload": "{
    "identifiers": [
      {
        "type": "dns",
        "value": "www.f5labs.local"
      }
    ]
  }
  -------------------------------------------
  HTTP 201
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Location: https://pebble.acmelabs.local:14000/my-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY
  Replay-Nonce: tG2LvEiAWlqayLL1Xd5p0A
  {
     "status": "pending",
     "expires": "2024-07-04T12:23:37Z",
     "identifiers": [
        {
           "type": "dns",
           "value": "www.f5labs.local"
        }
     ],
     "finalize": "https://pebble.acmelabs.local:14000/finalize-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY",
     "authorizations": [
        "https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0"
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
  POST https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "tG2LvEiAWlqayLL1Xd5p0A", 
        "url": "https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0"
    },
    "signature": "OLFcTqqJjkNiPRMES5romppnuA...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: LYvJr3PloWaEhjxpXwtJ2A
  {
     "status": "pending",
     "identifier": {
        "type": "dns",
        "value": "www.f5labs.local"
     },
     "challenges": [
        {
           "type": "dns-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/xuwc6c_dIoeN7SzdV6c4xB-i1B3ia3can6U3H55c8r8",
           "token": "H75UagVrBg9FK6f2rV2zBJiF2GX5jJa5tgJDaZb-MZo",
           "status": "pending"
        },
        {
           "type": "tls-alpn-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/FlVXCjrD43-pnYWUBsENWP8dI7Bab24GIiK0CE9tnq4",
           "token": "Iz3lA6KEaBqjCDfSE2ZtcTXsDxjcx2-ZlhC4aclfqhw",
           "status": "pending"
        },
        {
           "type": "http-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/j2JwAI_xnIjY3uez8MzXprPXcb4ghA94oC2F4Ih9mf4",
           "token": "LGbvLqb26AIdrnBnjDGfnu1ACE2zT_JUOobv6dCCfxY",
           "status": "pending"
        }
     ],
     "expires": "2024-07-03T13:23:37Z"
  }
  ```
</details>
<details>
  <summary>6. Client Function: Stage the HTTP port 80 listener and token</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target TLS resource. With the http-01 proof validation, the provider is going to query public DNS for the IP of the requested domain and then make an HTTP port 80 request to that domain seeking a response on the "/.well-known/acme-challenge/[token]" URL, expecting the token to be in the response. This proves to the provider that the requestor owns both the DNS record and the application asset. Using the --standalone option on the Certbot client enables it to stand up an ephemeral HTTP port 80 listener directly (without needing to manipulate the NGINX TLS server). This works because the Certbot client is running on the same server address as the NGINX instance. In a real-world scenario, however, it is often useful to manipulate the TLS server's configuration to have it listen on HTTP port 80 and respond with the validation token. This can be handled in several ways:

- Pre-stage the /.well-known/acme-challenge/ HTTP port 80 listener in the web server's configuration and then have the ACME client drop the token into that "folder" during the proof validation.
- Use a script to create a temporary HTTP port 80 listener config on the web server and insert the validation token.

Again, the best approach depends on the capabilities of the ACME client and target TLS resource. For example, the first option is technically easier and doesn't typically require a configuration reload, however would leave an HTTP port 80 listener open. The second option would require a configuration reload, but with a server like NGINX this isn't usually a problem.
  <br /><br />
</details>
<details>
  <summary>7. Let the provider know the challenge is ready</summary>
  <br />
  Notice also the "url" value in the http-01 block of the authorizations response. This URL is how the client will indicate its preference to use http-01 proof validation. The client needs to make a POST request to this URL, pass in "protected" block, empty "payload" block, and the "signature" block. The provider will return the same http-01 authorizations block with a "pending" status, indicating it will commence validation.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/chalZ/j2JwAI_xnIjY3uez8MzXprPXcb4ghA94oC2F4Ih9mf4
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "LYvJr3PloWaEhjxpXwtJ2A", 
        "url": "https://pebble.acmelabs.local:14000/chalZ/j2JwAI_xnIjY3uez8MzXprPXcb4ghA94oC2F4Ih9mf4"
    },
    "signature": "Oh6Y0dRxESsNNZd4byOgdWt9lt...",
    "payload": "{}"
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index", <https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0>;rel="up"
  Replay-Nonce: 5WLGGTY2Q62HtrUbkLYGmg
  {
     "type": "http-01",
     "url": "https://pebble.acmelabs.local:14000/chalZ/j2JwAI_xnIjY3uez8MzXprPXcb4ghA94oC2F4Ih9mf4",
     "token": "LGbvLqb26AIdrnBnjDGfnu1ACE2zT_JUOobv6dCCfxY",
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
  POST https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "5WLGGTY2Q62HtrUbkLYGmg", 
        "url": "https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0"
    },
    "signature": "ED0Fz2woEHf2-rod3h4g5e82-1...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: 2M9cK4dU0FY4viOVSjmV8A
  {
     "status": "valid",
     "identifier": {
        "type": "dns",
        "value": "www.f5labs.local"
     },
     "challenges": [
        {
           "type": "http-01",
           "url": "https://pebble.acmelabs.local:14000/chalZ/j2JwAI_xnIjY3uez8MzXprPXcb4ghA94oC2F4Ih9mf4",
           "token": "LGbvLqb26AIdrnBnjDGfnu1ACE2zT_JUOobv6dCCfxY",
           "status": "valid",
           "validated": "2024-07-03T12:23:37Z"
        }
     ],
     "expires": "2024-07-03T13:23:37Z"
  }
  ```
</details>
<details>
  <summary>9. Client Function: Clean up the HTTP port 80 listener and token</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target TLS resource. For http-01, this simply means either removing the ephemeral validation token from the HTTP port 80 listener, or removing the HTTP port 80 listener completely.
  <br /><br >
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
  POST https://pebble.acmelabs.local:14000/finalize-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "2M9cK4dU0FY4viOVSjmV8A", 
        "url": "https://pebble.acmelabs.local:14000/finalize-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY"
    },
    "signature": "F9Oy8VYG4NZyE9P5JgV8Q8s1fi...",
    "payload": {
        "csr": "MIHqMIGQAgEAMAAwWTATBgcqhk..."
     }
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Location: https://pebble.acmelabs.local:14000/my-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY
  Replay-Nonce: m9uZCc-gDt_QsAuDlYt66w
  {
     "status": "processing",
     "expires": "2024-07-04T12:23:37Z",
     "identifiers": [
        {
           "type": "dns",
           "value": "www.f5labs.local"
        }
     ],
     "finalize": "https://pebble.acmelabs.local:14000/finalize-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY",
     "authorizations": [
        "https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0"
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
  POST https://pebble.acmelabs.local:14000/my-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "m9uZCc-gDt_QsAuDlYt66w", 
        "url": "https://pebble.acmelabs.local:14000/my-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY"
    },
    "signature": "PfZp4Erw9BsHOeRJsxeHyz3Rox...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  Replay-Nonce: LS4vJrt_AjcplOyPeHxTJw
  {
     "status": "valid",
     "expires": "2024-07-04T12:23:37Z",
     "identifiers": [
        {
           "type": "dns",
           "value": "www.f5labs.local"
        }
     ],
     "finalize": "https://pebble.acmelabs.local:14000/finalize-order/89IpX7w9L0vFmE-82kmUxE5X1r-8_lIORXOh2kNxrUY",
     "authorizations": [
        "https://pebble.acmelabs.local:14000/authZ/ypVmnrbvWsEjONv0ygSAPCkh7LaLSq0O98DqohHr0Y0"
     ],
     "certificate": "https://pebble.acmelabs.local:14000/certZ/730eee6d49373bab"
  }
  ```
</details>
<details>
  <summary>12. Retrieve Certificates</summary>
  <br />
  Once the provider returns the certificate URL, it can use this URL to fetch the new certificate. The provider will usually send both the renewed certificate and its issuer. The certificate(s) will be in PEM format.
  <br />
  
  ```
  POST https://pebble.acmelabs.local:14000/certZ/730eee6d49373bab
  {
    "protected": {
        "alg": "RS256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1", 
        "nonce": "LS4vJrt_AjcplOyPeHxTJw", 
        "url": "https://pebble.acmelabs.local:14000/certZ/730eee6d49373bab"
    },
    "signature": "PyZmOf9udEJYftlgVIX-hQ2VKL...",
    "payload": ""
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/pem-certificate-chain; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index", <https://pebble.acmelabs.local:14000/certZ/730eee6d49373bab/alternate/1>;rel="alternate"
  Replay-Nonce: 1-EDvoC_Lh1wHgr4QVBQMg
  Transfer-Encoding: chunked
  
  -----BEGIN CERTIFICATE-----
  MIICmDCCAYCgAwIBAgIIcw7ubUk3O6swDQYJKoZIhvcNAQELBQAwKDEmMCQGA1UE
  ...
  YWuTWjzfrmUVM37Pbeoru5tR+kW2LwLuKw+pkECuV4tBLq6L0mgy4Gk/RDk=
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  MIIDUDCCAjigAwIBAgIId5Z4J4JfGSYwDQYJKoZIhvcNAQELBQAwIDEeMBwGA1UE
  ...
  cL6yx5whyngs+a2EhHPLAe5sSCLLpGux7aAOsLKk+VSUsvwP
  -----END CERTIFICATE-----
  ```
</details>
<details>
  <summary>13. Client Function: Move the Certificate(s) into Position</summary>
  <br />
  Wherever the ACME client may be running, it now needs to move the new certificate(s) into position where the TLS server needs them. In the case of a server like NGINX, it also needs to reload the configuration data to update the certificates in memory. For the sake of completeness, this lab's Bash script is included that simply copies the renewed certificates into the location that NGINX is expecting, and then issues a config reload. This process is otherwise highly dependent on the TLS server.
  <br />
  
  ```
  #!/bin/bash

  ## Move the renewed certificate to the correct location for the NGINX config.
  cp -f /etc/letsencrypt/live/${RENEWED_DOMAINS}/* /etc/letsencrypt/live/f5labs.local/

  ## Reload the NGINX config
  nginx -s reload
  ```
</details>

While there are variations not discussed here (ex. key rotation), the above summarizes a generic ACME protocol exchange for http-01 proof validation.

