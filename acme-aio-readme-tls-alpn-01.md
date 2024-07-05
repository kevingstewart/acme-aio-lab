### ACMEv2 All-in-One Testing Lab for TLS-ALPN-01 Proof Validation
All ACME proof validations must prove ownership of *something*. For the **tls-alpn-01** challenge, the client must prove:

- Ownership of the DNS domain
- Ownership and control of the TLS configuration of the application asset

<br />

That is, the client must own the DNS record for the requested domain, and must have control of the TLS configuration of the TLS service that the DNS record point to. The following description of the protocol exchange is super-simplistic for the sake of illustrating the tls-alpn-01 proof requirement, and assumes things like client registration are complete. A more detailed protocol exchange is included further down in this document. Also for the sake of this document, the term "ACME provider", or simply "provider" is used to indicate an ACME service (ex. Let's Encrypt).


#### References
ACMEv2 tls-alpn-01 is covered in the following references:
- https://www.rfc-editor.org/rfc/rfc8737.html

<br />


#### Simplified ACME TLS-ALPN-01 Protocol Exchange
- An ACME client sends a request to an ACME provider to "order" a new certificate (ex. www.example.com).
- The ACME provider sends back a list of supported proof validation options, typically **http-01**, **dns-01**, and **tls-alpn-01**, and the client indicates that it wants to use **tls-alpn-01**.
- The provider then sends a validation token (ex. _V8HttT5ph9UNpcIM0sF28B7dfFVtdH7P_)
- The client must now configures the TLS properties of the TLS application to inject the token into the ALPN attributes so that when the provider queries, the validation token will be in the ALPN portion of the TLS handshake. The client then tells the provider that it's ready.
- The provider may take some time to perform its validation, so the client will periodically poll the provider for status. Once the provider is done and the status is good, the client will send a Certificate Signing Request (CSR) for the requested certificate.
- The provider will create the certificate and send the client a URL it can use to fetch it.
- The client fetches the new certificate from this URL and should now remove HTTP listener configuration.

This is the general flow of ACMEv2 tls-alpn-01. Again, a more detailed and correct description is added below for reference. Once the client has access to the new certificate it may need to push that to the TLS service that actually needs it. The provisions for this step are specific to the TLS service. Also, in a real world scenario, the DNS is some Internet-based entity (ex. Cloudflare, Google), so the client may have no control over this service, and only have control over the actual target application asset. This is the essential premise of the tls-alpn-01 ACME proof validation.

<br />

-----
### Testing ACMEv2 HTTP-01 in the All-in-One Lab Environment
TBD

<br />

-----
### Detailed ACMEv2 HTTP-01 Protocol Exchange
TBD
