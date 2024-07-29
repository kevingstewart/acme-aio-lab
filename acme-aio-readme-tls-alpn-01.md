### ACMEv2 All-in-One Testing Lab for TLS-ALPN-01 Proof Validation
All ACME proof validations must prove ownership of *something*. For the **tls-alpn-01** challenge, the client must prove:

- Ownership of the DNS domain
- Ownership and control of the TLS configuration of the application asset

<br />

That is, the client must own the DNS record for the requested domain, and must have control of the TLS configuration of the TLS service that the DNS record points to. The following description of the protocol exchange is super-simplistic for the sake of illustrating the tls-alpn-01 proof requirement, and assumes things like client registration are complete. A more detailed protocol exchange is included further down in this document. Also for the sake of this document, the term "ACME provider", or simply "provider" is used to indicate an ACME service (ex. Let's Encrypt).


#### References
ACMEv2 tls-alpn-01 is covered in the following references:
- https://www.rfc-editor.org/rfc/rfc8737.html

<br />


#### Simplified ACME TLS-ALPN-01 Protocol Exchange
- An ACME client sends a request to an ACME provider to "order" a new certificate (ex. www.example.com).
- The ACME provider sends back a list of supported proof validation options, typically **http-01**, **dns-01**, and **tls-alpn-01**, and the client indicates that it wants to use **tls-alpn-01**.
- The provider then sends a validation token (ex. _V8HttT5ph9UNpcIM0sF28B7dfFVtdH7P_)
- The client must now configures the TLS properties of the TLS application to inject the token in the TLS handshake. The client then tells the provider that it's ready. In the case of tls-alpn-01, the client needs to do two things:
  - Configure a TLS listener to support an ALPN extension with the single protocol name "acme-tls/1".
  - Create a self-signed certificate containing the single requested domain name as the subjectAltName extension, and an "acmeIdentifier" extension containing the SHA-256 digest of the authorization key.
- The provider may take some time to perform its validation, so the client will periodically poll the provider for status. Once the provider is done and the status is good, the client will send a Certificate Signing Request (CSR) for the requested certificate.
- The provider will create the certificate and send the client a URL it can use to fetch it.
- The client fetches the new certificate from this URL and should now remove TLS validation configuration.

This is the general flow of ACMEv2 tls-alpn-01. Again, a more detailed and correct description is added below for reference. Once the client has access to the new certificate it may need to push that to the TLS service that actually needs it. The provisions for this step are specific to the TLS service. Also, in a real world scenario, the DNS is some Internet-based entity (ex. Cloudflare, Google), so the client may have no control over this service, and only have control over the actual target application asset. This is the essential premise of the tls-alpn-01 ACME proof validation.

<br />

-----
### Testing ACMEv2 TLS-ALPN-01 in the All-in-One Lab Environment
The ACMEv2 all-in-one lab consists of a Docker Compose file that builds all of the necessary components to support a fully-contained, container-based environment. No external services are required. For TLS-ALPN-01 testing specifically, you need an ACME client (Netshoot server with acme.sh client), an ACME provider (Pebble or Smallstep), and a DNS server (Bind). During the proof phase of the protocol exchange, the client must be able to answer the ACME challenge through a TLS handshake from the provider. Note that the NGINX server in this lab is already listening on TLS port 443. It is technically possible to dynamically reconfigure the NGINX server to handle the tls-alpn-01 challenges, but for simplicity the lab will use a "standalone" feature of the acme.sh client, so must be tested from the separate Netshoot host.

<br />

**To test tls-alpn-01**:

1. Start the Docker Compose environment.
   ```shell
   docker compose -f acme-aio-internal-compose.yaml up -d
   ```
2. Tail the Netshoot container log until the logs settles. Many things are happening under the hood.
   ```shell
   docker logs -f netshoot
   ```
3. Shell into the Netshoot container.
   ```shell
   docker exec -it netshoot /bin/bash
   ```
4. From the Netshoot container, execute an ACME certificate renewal request to one of the ACME providers for a new certificate. The --debug option on the command line will dump the entire ACME protocol exchange for your review. The below example uses the Pebble ACME server for the utility.f5labs.local domain.
   ```shell
   export SERVER=https://pebble.acmelabs.local:14000/dir
   #export SERVER=https://smallstep.acmelabs.local:9000/acme/acme/directory
   export DOMAIN=utility.f5labs.local
   /root/acme/acme.sh --issue --alpn --insecure --server ${SERVER} -d ${DOMAIN} --keylength 2048 --output-insecure --debug --force
   ```
5. From the Netshoot container, view the properties of the renewed certificate.
   ```shell
   openssl x509 -noout -text -in /root/.acme.sh/utility.f5labs.local/utility.f5labs.local.cer
   ```
6. Optionally, shut down the Docker Compose when you are done testing. This will reset all configuration data.
   ```shell
   docker compose down
   ```

<br />

-----
### Detailed ACMEv2 TLS-ALPN-01 Protocol Exchange
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
  Replay-Nonce: V4U6YF4fQ1kIqlfDRtM4AQ
  ```
</details>

<details>
  <summary>3. Registration request (newAccount service)</summary>
  <br />
  Assuming the client has not yet registered with the ACME provider, it needs to first make a POST request to the "newAccount" service. The content of the request payload includes a "payload" block containing the "contact" email address and agreement to the provider's terms-of-service, a "protected" block that contains the previous nonce, service URL, and JSON web key attributes (algorithm, key type, modulus[n], and exponent[e]), and a "signature" block that is a digital signature using the client's private key. Note that in this and all following requests, the "protected" and "payload" blocks are base64-encoded. These are shown decoded here to better understand the protocol exchange. Also note that the provider should return a new nonce value in each response, which the client should use in the subsequent request.
   
  ```
  POST https://pebble.acmelabs.local:14000/sign-me-up
  {
    "protected": {
      "nonce": "Zg-afyqnKaoaral12ifuRA", 
      "url": "https://pebble.acmelabs.local:14000/sign-me-up", 
      "alg": "ES256", 
      "jwk": {
        "crv": "P-256", 
        "kty": "EC", 
        "x": "rNxQYtY7fF_AxCycllVc6zNvuDbv3KXVAk5WYDS-Fxg", 
        "y": "JVLY5pBd_Ok8Jtwmo38tSS5FfJjAw2QxHm83-ijowkw"
      }
    }, 
    "payload": {
      "termsOfServiceAgreed": true
    },
    "signature": "..."
  }
  -------------------------------------------
  {
    "status": "valid",
    "orders": "https://pebble.acmelabs.local:14000/list-orderz/1",
    "key": {
       "kty": "EC",
       "crv": "P-256",
       "x": "rNxQYtY7fF_AxCycllVc6zNvuDbv3KXVAk5WYDS-Fxg",
       "y": "JVLY5pBd_Ok8Jtwmo38tSS5FfJjAw2QxHm83-ijowkw"
    }
  }
  ```
  
  <br />
  Critical to the tls-alpn-01 proof validation, the client is in possession of a JSON web key (jwk) that is to be part of the acmeIdentifier value in the TLS certificate. The "account thumbrint" is created by performing a SHA256 digest of this jwk:

  ```
  jwk='{"crv": "P-256", "kty": "EC", "x": "rNxQYtY7fF_AxCycllVc6zNvuDbv3KXVAk5WYDS-Fxg", "y": "JVLY5pBd_Ok8Jtwmo38tSS5FfJjAw2QxHm83-ijowkw"}'
  ACCOUNT_THUMBPRINT=printf "%s" "$jwk" | tr -d ' ' | openssl dgst -sha256 -binary | base64 | tr '/+' '_-' | tr -d '= '

  --> returns "ADIKWIdBiFhd364g1tTY0YcmImSVayKIOX7obdTStNw"
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
        "nonce": "nX2AF3i34tTI8wFUFN7CCQ", 
        "url": "https://pebble.acmelabs.local:14000/order-plz", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": {
        "identifiers": [
            {
                "type":"dns",
                "value":"utility.f5labs.local"
            }
        ]
    },
    "signature": "..."
  }
  -------------------------------------------
  HTTP 201
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  Link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  location: https://pebble.acmelabs.local:14000/my-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA
  replay-nonce: L6XENUNT0b7LOX9snCaoJA
  {
    "status": "pending",
    "expires": "2024-07-18T20:16:19Z",
    "identifiers": [
      {
        "type": "dns",
        "value": "utility.f5labs.local"
      }
    ],
    "finalize": "https://pebble.acmelabs.local:14000/finalize-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA",
    "authorizations": [
      "https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo"
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
  POST https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo
  {
    "protected": {
        "nonce": "L6XENUNT0b7LOX9snCaoJA", 
        "url": "https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": "", 
    "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  replay-nonce: suiDGTLRH24J0njO75-wyA
  {
    "status": "pending",
    "identifier": {
      "type": "dns",
      "value": "utility.f5labs.local"
    },
    "challenges": [
      {
        "type": "tls-alpn-01",
        "url": "https://pebble.acmelabs.local:14000/chalZ/Y8R1GatF70iP14sjI4C6oUc8b8HXBLKSJ-3-k-UdyhM",
        "token": "pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU",
        "status": "pending"
      },
      {
        "type": "dns-01",
        "url": "https://pebble.acmelabs.local:14000/chalZ/J5E71QY8CIzQ-XziOgAoJxoESxdr3dH6UBu21CENFkI",
        "token": "Cj2AfIg_HxQnxwG8ntmpT405f6WqjYqjhHKETxnYy1M",
        "status": "pending"
      },
      {
        "type": "http-01",
        "url": "https://pebble.acmelabs.local:14000/chalZ/JQqtr9yqVG8ENZ6p_JGsD9OnzqAlL0O8C8Z2_Cmr-m4",
        "token": "kh9-T3rMb6Vw26LhYCK2a19zpHWeWEwaVBQKEtFXuNo",
        "status": "pending"
      }
    ],
    "expires": "2024-07-17T21:16:19Z"
  }
  ```

  <br />
  The client now has the validation token for tls-alpn-01 (pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU) and can use that to complete the "authorization key" to be used in the acmeIdentifier extension of the certificate. The authorization key is the SHA256 digest of the "[TOKEN].[ACCOUNT_THUMBPRINT]" values:
  <br /><br />

  ```
  ACCOUNT_THUMBPRINT="ADIKWIdBiFhd364g1tTY0YcmImSVayKIOX7obdTStNw"
  TOKEN="pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU"
  AUTHORIZATION_KEY="pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU.ADIKWIdBiFhd364g1tTY0YcmImSVayKIOX7obdTStNw"

  printf "%s" "$AUTHORIZATION_KEY" | openssl dgst -sha256 -hex

  --> returns "78364af0434f189d288f20f4fb4676552766f21683113ec6ffd7b663519a1837"
  ```
</details>

<details>
  <summary>6. Client Function: Stage the TLS ALPN challenge</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target TLS resource. The goal is to insert the challenge token into the TLS ALPN configuration of the application. Proof validation is established by virtue of the TLS handshake that happens from the provider the the TLS application. For the sake of completeness, however, the acme.sh client in this lab will not modify the NGINX TLS application, but rather use a standalone function on the Netshoot host. The acme.sh client will create the requisite TLS ALPN configuration and start a TLS port 443 listener directly.
  <br /><br />
  The tls-alpn-01 proof validation requires a TLS server configured to use an ALPN extension with a single protocol name "acme-tls/1" and a self-signed validation certificate containing the single requested domain name as the subjectAltName extension, and the (ASN.1 DER-encoded) SHA256 digest authorization key as an acmeIdentifier extension (1.3.6.1.5.5.7.1.31: critical). The provider will initiate a TLS ALPN handshake with the server and expect to see these values. Please refer to https://www.rfc-editor.org/rfc/rfc8737.html for exact technical semantics. The below is an example tls-alpn-01 validation certificate. The important fields are: 
  <br /><br />
   
   - The "X509v3 Subject Alternative Name" extension indicating the requested domain name
   - The "1.3.6.1.5.5.7.1.31: critical" extension indicating the "acmeIdentifier" - an ASN.1 DER-encoded SHA256 digest of the authorization key

   ```
   Certificate:
       Data:
           Version: 3 (0x2)
           Serial Number:
               68:4d:fc:10:eb:8b:55:78:39:9e:d8:65:63:45:2c:fa:3b:5c:dc:9f
           Signature Algorithm: sha256WithRSAEncryption
           Issuer: CN=tls.acme.sh
           Validity
               Not Before: Jul 18 13:18:33 2024 GMT
               Not After : Jul 18 13:18:33 2025 GMT
           Subject: CN=tls.acme.sh
           Subject Public Key Info:
               Public Key Algorithm: rsaEncryption
                   Public-Key: (2048 bit)
                   Modulus:
                       00:a7:af:fb:da:25:fe:91:45:d0:08:6e:85:5e:f6:
                       2d:32:63:0f:1a:96:3b:22:db:45:69:7a:24:87:76:
                       f5:0d:61:5f:91:b4:c9:30:5a:bb:7b:1c:83:6d:e4:
                       0a:0a:b4:56:11:21:90:2b:09:47:2c:7c:ce:6c:42:
                       bc:6e:e4:5c:26:1c:96:41:85:15:ce:c1:b2:1b:10:
                       a6:13:af:27:9b:ce:75:f4:5e:cd:b8:32:91:7e:de:
                       34:54:18:7f:cb:93:71:4d:87:aa:71:0c:04:4b:ac:
                       4e:07:04:31:0a:6b:84:7e:fa:af:68:0d:42:61:79:
                       e2:75:14:75:bb:dd:50:0b:44:f0:0f:2e:70:2d:0c:
                       d2:e6:ca:3d:3a:b3:ef:50:d6:8c:b6:21:f2:4c:e3:
                       c3:e8:a7:2d:f2:a4:ef:49:9c:9b:93:e4:e8:16:12:
                       24:3a:8a:0a:99:e7:bd:4b:d6:ab:f6:e3:83:6e:9a:
                       f4:0d:ac:cf:5c:ab:1b:01:15:56:b3:6a:da:3e:21:
                       0e:f8:d4:a0:8b:c5:40:7e:6e:1f:4c:89:97:f3:f4:
                       3b:ff:3a:fb:5d:d8:46:8d:8d:ad:39:a0:00:de:1d:
                       f7:3c:aa:eb:82:19:c9:9a:48:4f:15:57:ef:dd:d6:
                       7f:69:48:49:fc:c1:82:ea:ef:7b:c6:c0:48:6d:ea:
                       b2:2b
                   Exponent: 65537 (0x10001)
           X509v3 extensions:
               X509v3 Extended Key Usage: 
                   TLS Web Server Authentication, TLS Web Client Authentication
               X509v3 Subject Alternative Name: 
                   DNS:utility.f5labs.local
               1.3.6.1.5.5.7.1.31: critical
                   . ....<.R#x2E...)LA..A...o....w...
               X509v3 Subject Key Identifier: 
                   76:6B:DF:80:BC:7C:60:D8:A2:61:84:35:E5:23:3A:F3:3E:97:F9:78
       Signature Algorithm: sha256WithRSAEncryption
       Signature Value:
           84:14:c7:02:e2:b0:d5:93:12:be:b3:c7:80:e7:36:35:7e:84:
           a9:1b:a1:55:d9:c6:25:ab:ca:87:b8:23:5f:70:d8:1b:25:e3:
           7c:fe:70:8a:2a:4b:54:97:9f:17:ff:96:64:52:eb:f9:3b:dd:
           ce:f3:03:38:3c:67:81:dc:63:67:b4:3f:09:81:fd:df:48:e9:
           6d:9a:d9:de:3d:84:54:52:cf:76:8a:05:0c:3e:0e:0a:b6:ab:
           f5:c2:68:c9:01:40:51:f5:b6:4c:c9:23:87:70:c1:d3:25:7b:
           29:50:39:1f:f6:f5:60:d6:24:63:41:7e:de:b8:de:4a:78:f0:
           f7:07:ff:2c:a8:b1:0f:bb:ea:3c:eb:07:4d:76:d5:b0:8a:21:
           21:2a:ea:47:a4:72:26:70:9a:fc:ef:b2:ad:4a:fe:50:89:1b:
           60:66:67:3f:6a:2c:d1:fb:f0:d2:cf:35:f8:20:66:17:9a:16:
           48:f1:23:e1:46:da:ca:52:8b:07:ed:87:be:c0:eb:8f:d8:26:
           5a:aa:d5:cf:ec:0f:99:a6:d7:4d:f4:f0:98:b8:84:ea:fe:c8:
           33:2f:af:3c:42:4b:67:c5:03:b3:a5:64:7c:95:cd:26:4f:d9:
           f0:95:5e:1a:fc:db:2b:70:a4:48:e1:7b:a2:60:d8:86:52:f7:
           53:51:ee:0c
   ```
</details>

<details>
  <summary>7. Let the provider know the challenge is ready</summary>
  <br />
  Notice also the "url" value in the tls-alpn-01 block of the authorizations response. This URL is how the client will indicate its preference to use tls-alpn-01 proof validation. The client needs to make a POST request to this URL, pass in "protected" block, empty "payload" block, and the "signature" block. The provider will return the same tls-alpn-01 authorizations block with a "pending" status, indicating it will commence validation.
  <br />

  ```
  POST https://pebble.acmelabs.local:14000/chalZ/Y8R1GatF70iP14sjI4C6oUc8b8HXBLKSJ-3-k-UdyhM
  {
    "protected": {
        "nonce": "suiDGTLRH24J0njO75-wyA", 
        "url": "https://pebble.acmelabs.local:14000/chalZ/Y8R1GatF70iP14sjI4C6oUc8b8HXBLKSJ-3-k-UdyhM", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": {}, 
    "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo>;rel="up"
  replay-nonce: 8T-QQ4fv-BvaDg7L20uZ4g
  {
    "type": "tls-alpn-01",
    "url": "https://pebble.acmelabs.local:14000/chalZ/Y8R1GatF70iP14sjI4C6oUc8b8HXBLKSJ-3-k-UdyhM",
    "token": "pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU",
    "status": "pending"
  }
  ```
</details>

<details>
  <summary>8. Poll the provider for validation status</summary>
  <br />
  For the tls-alpn-01 proof, the ACME provider makes a TLS handshake to the application and will insert the following ALPN extension information into its Client Hello message. Note the "acme-tls/1" string.

  ```
  Extension: application_layer_protocol_negotiation (len=13)
    Type: application_layer_protocol_negotiation (16)
    Length: 13
    ALPN Extension Length: 11
    ALPN Protocol
        ALPN string length: 10
        ALPN Next Protocol: acme-tls/1
  ```

  On receipt of this Client Hello the application must use the ephemeral verification certification to complete the TLS handshake. A busy ACME provider may take some time to get to this validation, so the client should continue to poll the provider for status. To do that it makes a POST request to the same authorizations URL, passing in "protected" block, empty "payload" block, and the "signature" block. Once the provider has had a chance to validate the challenge (initiate a TLS handshake) it will return a response to the client's poll indicating a "valid" status.
  <br />

  ```
  POST https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo
  {
    "protected": {
        "nonce": "8T-QQ4fv-BvaDg7L20uZ4g", 
        "url": "https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": "", 
    "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  replay-nonce: 13KZ1OdFr-KQJw-uau6P7w
  {
    "status": "valid",
    "identifier": {
      "type": "dns",
      "value": "utility.f5labs.local"
    },
    "challenges": [
      {
        "type": "tls-alpn-01",
        "url": "https://pebble.acmelabs.local:14000/chalZ/Y8R1GatF70iP14sjI4C6oUc8b8HXBLKSJ-3-k-UdyhM",
        "token": "pahg-IDA0PtJku-7oJJHmCJu3kee9Kdmq4Jf84SPzjU",
        "status": "valid",
        "validated": "2024-07-17T20:16:21Z"
      }
    ],
    "expires": "2024-07-17T21:16:21Z"
  }
  ```
</details>

<details>
  <summary>9. Client Function: Clean up the TLS ALPN challenge</summary>
  <br />
  The implementation of this step is dependent on both the client's capabilities and the target TLS application. The goal here is simply to remove the ALPN "acme-tls/1" configuration and corresponding self-signed validation certificate from the server's TLS settings. 
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
  POST https://pebble.acmelabs.local:14000/finalize-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA
  {
    "protected": {
        "nonce": "13KZ1OdFr-KQJw-uau6P7w", 
        "url": "https://pebble.acmelabs.local:14000/finalize-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": {
        "csr": "MIHpMIGQAgEA..."
    },
    "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  location: https://pebble.acmelabs.local:14000/my-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA
  replay-nonce: _id8VmHZybwm0lBBa6L3wA
  {
    "status": "processing",
    "expires": "2024-07-18T20:16:19Z",
    "identifiers": [
      {
        "type": "dns",
        "value": "utility.f5labs.local"
      }
    ],
    "finalize": "https://pebble.acmelabs.local:14000/finalize-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA",
    "authorizations": [
      "https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo"
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
  POST https://pebble.acmelabs.local:14000/my-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA
  {
      "protected": {
          "nonce": "_id8VmHZybwm0lBBa6L3wA", 
          "url": "https://pebble.acmelabs.local:14000/my-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA", 
          "alg": "ES256", 
          "kid": "https://pebble.acmelabs.local:14000/my-account/1"
      },
      "payload": "", 
      "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/json; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  replay-nonce: 4v6TmsDAal5UetlJtQ-B9w
  {
    "status": "valid",
    "expires": "2024-07-18T20:16:19Z",
    "identifiers": [
      {
        "type": "dns",
        "value": "utility.f5labs.local"
      }
    ],
    "finalize": "https://pebble.acmelabs.local:14000/finalize-order/DmPPhj0enriu3POHtxys5-O_4MtZMle4B42-576C8lA",
    "authorizations": [
      "https://pebble.acmelabs.local:14000/authZ/2UiaScz6g6kxJtoZ5EWyqvgaqmJ8BpQx47nfUuhC6Wo"
    ],
    "certificate": "https://pebble.acmelabs.local:14000/certZ/24921bcbcd5ed6a3"
  }
  ```  
</details>

<details>
  <summary>12. Retrieve Certificates</summary>
  <br />
  Once the provider returns the certificate URL, it can use this URL to fetch the new certificate. The provider will usually send both the renewed certificate and its issuer. The certificate(s) will be in PEM format.
  <br />

  ```
  POST https://pebble.acmelabs.local:14000/certZ/24921bcbcd5ed6a3
  {
    "protected": {
        "nonce": "4v6TmsDAal5UetlJtQ-B9w", 
        "url": "https://pebble.acmelabs.local:14000/certZ/24921bcbcd5ed6a3", 
        "alg": "ES256", 
        "kid": "https://pebble.acmelabs.local:14000/my-account/1"
    },
    "payload": "", 
    "signature": "..."
  }
  -------------------------------------------
  HTTP 200
  Cache-Control: public, max-age=0, no-cache
  Content-Type: application/pem-certificate-chain; charset=utf-8
  link: <https://pebble.acmelabs.local:14000/dir>;rel="index"
  link: <https://pebble.acmelabs.local:14000/certZ/24921bcbcd5ed6a3/alternate/1>;rel="alternate"
  replay-nonce: -ySjA1pyflHxKVSnEvK3mg
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
  Wherever the ACME client may be running, it now needs to move the new certificate(s) into position where the TLS server needs them. In the case of a server like NGINX, it also needs to reload the configuration data to update the certificates in memory. This lab uses the standalone function of the acme.sh client on the Netshoot host, so none of that is required here. The resulting certificates are simply dropped into the acme.sh directory path.
  <br />
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

















