# ACMEv2 All-in-One Testing Lab
An ACMEv2 all-in-one testing lab that supports http-01, dns-01, and tls-alpn-01 challenges

### Introduction
ACME is an *automated* certificate renewal protocol that relies on "proof of ownership of **something**". While it is intended to be extensible, the standard ACMEv2 (version 2) implementation generally supports three forms of proof validation:

* [DNS (dns-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-dns-01.md) - proof of ownership of a domain
* [HTTP (http-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-http-01.md) - proof of ownership of a domain and application asset
* [TLS (tls-alpn-01)](https://github.com/kevingstewart/acme-aio-lab/blob/main/acme-aio-readme-tls-alpn-01.md) - proof of ownership of a domain and TLS application asset (pending)...

The above linked pages describe each proof validation in more detail, as a series of client-server challenge and response functions. And within these pages, specific testing scenarios are also covered. The purpose of this repository is to demonstrate the inner workings of the ACMEv2 protocol, within a fully self-contained, container-based, client-server ACMEv2 testing lab. No external resources are required (ex. DNS, Let's Encrypt, etc.) to operate this lab.

<br />

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

<br />

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

<br />

----

### Technical Notes
Below is a compilation of additional relevant notes on ACMEv2 implementation.

<details>
  <summary>acme.sh client command options</summary>

  ```
  https://github.com/acmesh-official/acme.sh
  v3.0.8
  Usage: acme.sh <command> ... [parameters ...]
  Commands:
    -h, --help               Show this help message.
    -v, --version            Show version info.
    --install                Install acme.sh to your system.
    --uninstall              Uninstall acme.sh, and uninstall the cron job.
    --upgrade                Upgrade acme.sh to the latest code from https://github.com/acmesh-official/acme.sh.
    --issue                  Issue a cert.
    --deploy                 Deploy the cert to your server.
    -i, --install-cert       Install the issued cert to Apache/nginx or any other server.
    -r, --renew              Renew a cert.
    --renew-all              Renew all the certs.
    --revoke                 Revoke a cert.
    --remove                 Remove the cert from list of certs known to acme.sh.
    --list                   List all the certs.
    --info                   Show the acme.sh configs, or the configs for a domain with [-d domain] parameter.
    --to-pkcs12              Export the certificate and key to a pfx file.
    --to-pkcs8               Convert to pkcs8 format.
    --sign-csr               Issue a cert from an existing csr.
    --show-csr               Show the content of a csr.
    -ccr, --create-csr       Create CSR, professional use.
    --create-domain-key      Create an domain private key, professional use.
    --update-account         Update account info.
    --register-account       Register account key.
    --deactivate-account     Deactivate the account.
    --create-account-key     Create an account private key, professional use.
    --install-cronjob        Install the cron job to renew certs, you don't need to call this. The 'install' command can automatically install the cron job.
    --uninstall-cronjob      Uninstall the cron job. The 'uninstall' command can do this automatically.
    --cron                   Run cron job to renew all the certs.
    --set-notify             Set the cron notification hook, level or mode.
    --deactivate             Deactivate the domain authz, professional use.
    --set-default-ca         Used with '--server', Set the default CA to use.
                             See: https://github.com/acmesh-official/acme.sh/wiki/Server
    --set-default-chain      Set the default preferred chain for a CA.
                             See: https://github.com/acmesh-official/acme.sh/wiki/Preferred-Chain
  
  
  Parameters:
    -d, --domain <domain.tld>         Specifies a domain, used to issue, renew or revoke etc.
    --challenge-alias <domain.tld>    The challenge domain alias for DNS alias mode.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode
  
    --domain-alias <domain.tld>       The domain alias for DNS alias mode.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/DNS-alias-mode
  
    --preferred-chain <chain>         If the CA offers multiple certificate chains, prefer the chain with an issuer matching this Subject Common Name.
                                        If no match, the default offered chain will be used. (default: empty)
                                        See: https://github.com/acmesh-official/acme.sh/wiki/Preferred-Chain
  
    --valid-to    <date-time>         Request the NotAfter field of the cert.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/Validity
    --valid-from  <date-time>         Request the NotBefore field of the cert.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/Validity
  
    -f, --force                       Force install, force cert renewal or override sudo restrictions.
    --staging, --test                 Use staging server, for testing.
    --debug [0|1|2|3]                 Output debug info. Defaults to 2 if argument is omitted.
    --output-insecure                 Output all the sensitive messages.
                                        By default all the credentials/sensitive messages are hidden from the output/debug/log for security.
    -w, --webroot <directory>         Specifies the web root folder for web root mode.
    --standalone                      Use standalone mode.
    --alpn                            Use standalone alpn mode.
    --stateless                       Use stateless mode.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/Stateless-Mode
  
    --apache                          Use Apache mode.
    --dns [dns_hook]                  Use dns manual mode or dns api. Defaults to manual mode when argument is omitted.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/dnsapi
  
    --dnssleep <seconds>              The time in seconds to wait for all the txt records to propagate in dns api mode.
                                        It's not necessary to use this by default, acme.sh polls dns status by DOH automatically.
    -k, --keylength <bits>            Specifies the domain key length: 2048, 3072, 4096, 8192 or ec-256, ec-384, ec-521.
    -ak, --accountkeylength <bits>    Specifies the account key length: 2048, 3072, 4096
    --log [file]                      Specifies the log file. Defaults to "/root/.acme.sh/acme.sh.log" if argument is omitted.
    --log-level <1|2>                 Specifies the log level, default is 2.
    --syslog <0|3|6|7>                Syslog level, 0: disable syslog, 3: error, 6: info, 7: debug.
    --eab-kid <eab_key_id>            Key Identifier for External Account Binding.
    --eab-hmac-key <eab_hmac_key>     HMAC key for External Account Binding.
  
  
    These parameters are to install the cert to nginx/Apache or any other server after issue/renew a cert:
  
    --cert-file <file>                Path to copy the cert file to after issue/renew.
    --key-file <file>                 Path to copy the key file to after issue/renew.
    --ca-file <file>                  Path to copy the intermediate cert file to after issue/renew.
    --fullchain-file <file>           Path to copy the fullchain cert file to after issue/renew.
    --reloadcmd <command>             Command to execute after issue/renew to reload the server.
  
    --server <server_uri>             ACME Directory Resource URI. (default: https://acme.zerossl.com/v2/DV90)
                                        See: https://github.com/acmesh-official/acme.sh/wiki/Server
  
    --accountconf <file>              Specifies a customized account config file.
    --home <directory>                Specifies the home dir for acme.sh.
    --cert-home <directory>           Specifies the home dir to save all the certs, only valid for '--install' command.
    --config-home <directory>         Specifies the home dir to save all the configurations.
    --useragent <string>              Specifies the user agent string. it will be saved for future use too.
    -m, --email <email>               Specifies the account email, only valid for the '--install' and '--update-account' command.
    --accountkey <file>               Specifies the account key path, only valid for the '--install' command.
    --days <ndays>                    Specifies the days to renew the cert when using '--issue' command. The default value is 60 days.
    --httpport <port>                 Specifies the standalone listening port. Only valid if the server is behind a reverse proxy or load balancer.
    --tlsport <port>                  Specifies the standalone tls listening port. Only valid if the server is behind a reverse proxy or load balancer.
    --local-address <ip>              Specifies the standalone/tls server listening address, in case you have multiple ip addresses.
    --listraw                         Only used for '--list' command, list the certs in raw format.
    -se, --stop-renew-on-error        Only valid for '--renew-all' command. Stop if one cert has error in renewal.
    --insecure                        Do not check the server certificate, in some devices, the api server's certificate may not be trusted.
    --ca-bundle <file>                Specifies the path to the CA certificate bundle to verify api server's certificate.
    --ca-path <directory>             Specifies directory containing CA certificates in PEM format, used by wget or curl.
    --no-cron                         Only valid for '--install' command, which means: do not install the default cron job.
                                        In this case, the certs will not be renewed automatically.
    --no-profile                      Only valid for '--install' command, which means: do not install aliases to user profile.
    --no-color                        Do not output color text.
    --force-color                     Force output of color text. Useful for non-interactive use with the aha tool for HTML E-Mails.
    --ecc                             Specifies use of the ECC cert. Only valid for '--install-cert', '--renew', '--remove ', '--revoke',
                                        '--deploy', '--to-pkcs8', '--to-pkcs12' and '--create-csr'.
    --csr <file>                      Specifies the input csr.
    --pre-hook <command>              Command to be run before obtaining any certificates.
    --post-hook <command>             Command to be run after attempting to obtain/renew certificates. Runs regardless of whether obtain/renew succeeded or failed.
    --renew-hook <command>            Command to be run after each successfully renewed certificate.
    --deploy-hook <hookname>          The hook file to deploy cert
    --extended-key-usage <string>     Manually define the CSR extended key usage value. The default is serverAuth,clientAuth.
    --ocsp, --ocsp-must-staple        Generate OCSP-Must-Staple extension.
    --always-force-new-domain-key     Generate new domain key on renewal. Otherwise, the domain key is not changed by default.
    --auto-upgrade [0|1]              Valid for '--upgrade' command, indicating whether to upgrade automatically in future. Defaults to 1 if argument is omitted.
    --listen-v4                       Force standalone/tls server to listen at ipv4.
    --listen-v6                       Force standalone/tls server to listen at ipv6.
    --openssl-bin <file>              Specifies a custom openssl bin location.
    --use-wget                        Force to use wget, if you have both curl and wget installed.
    --yes-I-know-dns-manual-mode-enough-go-ahead-please  Force use of dns manual mode.
                                        See:  https://github.com/acmesh-official/acme.sh/wiki/dns-manual-mode
  
    -b, --branch <branch>             Only valid for '--upgrade' command, specifies the branch name to upgrade to.
    --notify-level <0|1|2|3>          Set the notification level:  Default value is 2.
                                        0: disabled, no notification will be sent.
                                        1: send notifications only when there is an error.
                                        2: send notifications when a cert is successfully renewed, or there is an error.
                                        3: send notifications when a cert is skipped, renewed, or error.
    --notify-mode <0|1>               Set notification mode. Default value is 0.
                                        0: Bulk mode. Send all the domain's notifications in one message(mail).
                                        1: Cert mode. Send a message for every single cert.
    --notify-hook <hookname>          Set the notify hook
    --notify-source <server name>     Set the server name in the notification message
    --revoke-reason <0-10>            The reason for revocation, can be used in conjunction with the '--revoke' command.
                                        See: https://github.com/acmesh-official/acme.sh/wiki/revokecert
  
    --password <password>             Add a password to exported pfx file. Use with --to-pkcs12.
  ```
</details>







