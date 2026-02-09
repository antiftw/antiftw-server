# AntiFTW.nl server setup for Docker 

## Contents

- [Instructions](#instructions)
  - [Initial setup](#main-setup)
  - [External Proxy (Hybrid Setup)](#external-proxy-hybrid-setup)
  - [TLS](TLS)
    - [LetsEncrypt](#letsencrypt)
	- [CaddyServer](#caddyserver)
	- [Self-Signed](#self-signed)
  - [Docker Mail Server](#docker-mail-server)
- [References](#references)

## Instructions

This repo contains the following `docker-compose.yml` fiiles to Dockerize my AntiFTW.nl server setup:

- `baikal/docker-compose.yml` - Card/Caldav server
- `caddy/docker-compose.yml` - Caddy Docker Proxy serving as reverse proxy
- `mailserver/docker-compose.yml` - Postfix Dovecot Emailserver (*)

(*) This will be downloaded when running `./mailserver/setup`, since it comes from a separate repository and needs some configuration. 

### Initial setup

Hence, first we need to run setup the mailserver:

```bash
cd mailserver
./setup mail.example.com [options]
```
The most important options will be shortly explained in this document. But for more information on the all available options and their usage run:

```bash
./setup --help
```
Then we need to create an external network so that Caddy can manage it:

```bash
docker network create caddy
```
And finally we can spin up the network by starting the different compose files in the seperate directories
```bash
docker compose -f caddy/docker-compose.yml up -d
docker compose -f baikal/docker-compose.yml up -d
docker compose -f mailserver/docker-compose.yml up -d
```

### External Proxy (Hybrid Setup)

The Caddy Docker Proxy can be extended with a custom Caddyfile to proxy external domains to other servers. This is useful when running multiple Pis on a home network with a single public IP.

**Architecture:**
```
Internet -> Router -> Pi5 (main gateway) -> Pi4 (testing server)
                           |
                           +-> Local Docker services (mail, dav, web, api)
```

The `caddy/Caddyfile` is mounted into the container and merged with the auto-generated config from caddy-docker-proxy. Edit this file to add external proxy rules:

```caddyfile
# Proxy to another server on the local network
mercure.dst-connect.io {
    reverse_proxy 192.168.50.97:443 {
        transport http {
            tls
            tls_insecure_skip_verify
        }
    }
}
```

**Note:** The target server (Pi4) manages its own TLS certificates. Pi5 terminates TLS for the incoming request and re-encrypts to Pi4. The `tls_insecure_skip_verify` is needed because Pi4's certificate is issued for the domain, not the internal IP.

### TLS

When running the `./setup` command, using the `-tls=` option we can indicate that you want to activate TLS (**strongly** recommended for a production environment), and select the method you would like to use. 

The currently supported options are:

- LetsEncrypt (le)
- CaddyServer (le-caddy)
- Self-signed (ss) (**NOT** recommended for production servers)

#### LetsEncrypt

Using this method you will have to manage and renew your certificates yourself. Create the certificates using Certbot, and the script will configure `docker-compose.yml` such that they are mounted such that they are available to DMS.

#### CaddyServer

This will configure CaddyServer (in this case the Caddy Docker Proxy server) in such a way so that it automatically generates certificates when starting the container for the first time. It will also renew them when necessary. This is the recommended setup.

⚠️ **Important** ⚠️ 

Due to some idiosyncrasy with Docker - where it will create new directories when mounting a non-existant file - in combination with the way CaddyServer generates the certificates, the script configures `docker-compose.yml` in such a way so that it **DOES NOT** mount the certificate files. 

So: we boot DMS the first time without these mounts, so that CaddyServer recognizes the domain and generates the certificate. After this, we need to shutdown DMS, and uncomment the lines beneath `# SSL_MOUNT` so that the certificate- and keyfile are mounted inside the container.

⚠️ **Important** ⚠️ 

#### Self-signed

**Warning:** Should only be used for testing and **NEVER** on a production envrironment. Therefore there is no automation. Instructions can be found in the documentation (See `TLS` in the [references](#references))

### SpamAssassin and Fail2Ban

These can be enabled when running the `./setup` command using either the `-sa` and `-f2b` options respectively. No further action is required to activate these services.

However, these can require addition configuration depending on your requirements. More information can be found in the [Fail2Ban](https://docker-mailserver.github.io/docker-mailserver/latest/config/security/fail2ban/) and [SpamAssassin](https://docker-mailserver.github.io/docker-mailserver/latest/faq/#spamassasin) docs.

### Docker Mail Server (DMS)

The mailserver requires some additional configuration. The most important steps are described in this document. To simplify setup, there is a setup command included with DMS. More information about its functionality can be found using:

```docker exec -it mailserver setup help```

The main things we want to setup are:

- Create at least one emailaddress using:

   ```docker exec -ti mailserver setup email add admin@example.com```
- Create at least one alias using:

	```docker exec -ti mailserver setup alias add postmaster@example.com admin@example.com```
- Setup DKIM, DMARC and SPF. More about this is the next sections.

#### DKIM
- Setting up DKIM (DomainKeys Identified mail) is pretty straightforward.

First we run 

```docker exec -ti mailserver setup config dkim```

And then we add the generated record to our domains DNS, located in: 

```./docker-data/dms/config/opendkim/keys/example.com/mail.txt```

For more information about the available options from the command we can use:

```docker exec -ti mailserver setup config dkim help```

#### DMARC

Since DMS is pre-configured with DMARC out of the box, this only consists of adding a DNS record. Something like this is a good initial record:

```_dmarc.example.com. IN TXT "v=DMARC1; p=none; sp=none; fo=0; adkim=4; aspf=r; pct=100; rf=afrf; ri=86400; rua=mailto:dmarc.report@example.com; ruf=mailto:dmarc.report@example.com"```

### SPF

This also only entails adding a DNS record, something like: 

```v=spf1 ip4:xxx.xxx.xxx.xxx ip6:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx:xxxx ~all```

Where we replace the ip addresses with the addresses from our server.


## References:

### Caddy

- Code completion for Caddy JSON config files - [caddy-json-schema](https://github.com/abiosoft/caddy-json-schema)
- Build custom caddy server - [XCaddy](https://github.com/caddyserver/xcaddy)
- Caddy LetsEncrypt staging - [Caddy.Community](https://caddy.community/t/how-to-use-lets-encrypt-staging-endpoint-with-caddy/18514)
- Caddy Docker Proxy - [Github](https://github.com/lucaslorentz/caddy-docker-proxy)
 
### Docker Mail Server
- [Github](https://github.com/antiftw/docker-mailserver)
- [General Documentation](https://docker-mailserver.github.io/docker-mailserver/latest/)
- [DKIM, DMARC and SPF](https://docker-mailserver.github.io/docker-mailserver/latest/config/best-practices/dkim_dmarc_spf/)
- [TLS](https://docker-mailserver.github.io/docker-mailserver/latest/config/security/ssl/)
- [Fail2Ban](https://docker-mailserver.github.io/docker-mailserver/latest/config/security/fail2ban/)
- [SpamAssassin](https://docker-mailserver.github.io/docker-mailserver/latest/faq/#spamassasin)

### Baikal

- [Baikal Docker](https://registry.hub.docker.com/r/ckulka/baikal/)