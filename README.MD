# AntiFTW.nl server setup for Docker 

## Contents

- [Instructions](#instructions)
- [Useful tips](#useful-tips)
- [References](#references)

## Instructions

This repo contains the following `docker-compose.yml` fiiles to Dockerize my AntiFTW.nl server setup:

- `baikal/docker-compose.yml` - Card/Caldav server
- `caddy/docker-compose.yml` - Caddy Docker Proxy serving as reverse proxy
- `mailserver/docker-compose.yml` - Postfix Dovecot Emailserver (*)

(*) This will be downloaded when running `./mailserver/setup`, since it comes from a separate repository and needs some configuration. 

Hence, first we need to run setup the mailserver:

```bash
cd mailserver
./setup <domain> [options]
```
For more information on the available options run:

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

The mailserver requires some additional configuration, which can be found through its [own repository](https://github.com/antiftw/docker-mailserver).

In short it comes down to the following steps:

- Create at least one emailaddress using 

   ```docker exec -ti <CONTAINER NAME> setup email add admin@example.com```

## Useful tips

### Code completion Caddy JSON Schema in VSCODE

Generate a Caddy server from scrath with included modules:

`xcaddy build v2.7.4     --with github.com/abiosoft/caddy-json-schema --with github.com/dunglas/mercure/caddy --with github.com/dunglas/vulcain/caddy`

Then Generate Caddy JSON Scheme for VSCODE, using the built Caddy binary including the modules:

`caddy json-schema --vscode`

This file will be written in a location such that VSCODE will automatically open it, and will automatically be applied to files called `*caddy*.[json|yaml]`.

### Use Letsencrypt staging endpoint in caddy.json file

```
{
	"apps": {
		"tls": {
			"automation": {
				"policies": [
					{
						"issuers": [
							{
								"module": "acme",
								"ca": "https://acme-staging-v02.api.letsencrypt.org/directory"
							}
						]
					}
				]
			}
		}
	}
}

```

## References:
 - Code completion for Caddy JSON config files - [caddy-json-schema](https://github.com/abiosoft/caddy-json-schema)
 - Build custom caddy server - [XCaddy](https://github.com/caddyserver/xcaddy)
 - Caddy LetsEncrypt staging - [Caddy.Community](https://caddy.community/t/how-to-use-lets-encrypt-staging-endpoint-with-caddy/18514)
 - Caddy Docker Proxy - [Github](https://github.com/lucaslorentz/caddy-docker-proxy)
 - Docker Mailserver - [Documentation](https://docker-mailserver.github.io/docker-mailserver/latest/)
 - Baikal Docker - [Github](https://registry.hub.docker.com/r/ckulka/baikal/)