---
layout:     post
title:      Setting up Synapse using <wbr>Docker&nbsp;Compose
cleanTitle: Setting up Synapse using Docker Compose
date:       2021-11-14 10:00:00
summary:    Setting up Matrix.org Synapse using Docker Compose and Traefik Proxy
categories: Computer
thumbnail: commenting
tags:
 - synapse
 - matrix
 - docker
 - docker compose
 - traefik
 - dns
---
In this post I'll outline the steps I took including setting up Synapse and what configuration I used. My install used simple username and password authentication and a 3rd party SMTP provider. I did not [set up a TURN server](https://matrix-org.github.io/synapse/latest/turn-howto.html) during my experience so that will not be included.

As Docker Compose is an extension of Docker syntax, all the steps involving it *can* be done without it however that will be an exercise left to the reader.

{% include toc.md %}

## Prerequisites 

If *(for some reason)* you are using this as a guide for doing this yourself, you will need the following:

### Knowledge
* *basic* Ubuntu Server and Linux
* *basic* Docker, Docker Compose
* *basic* Traefik Proxy

### Resources
* Internet-Accessible Server
	* Ubuntu 20.04 *or newer*
	* 4GB of RAM *minimum*
	* 8GB of Storage *minimum*
	* Terminal access
* Domain Name
	* Use of ports `443` and `8448`<sup>†<sup>
* Required Tools
	* [Docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)
	* [Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04)
	* [Traefik](https://doc.traefik.io/traefik/getting-started/quick-start/#launch-traefik-with-the-docker-provider) with Docker Provider
* Caffeine Delivery System *(optional)*

<sup><sup>†</sup>Alternatives described in [Domain Set-up](#domain-set-up)   
*Need a server? Get $100 in credit for 2 months through my [DigitalOcean Referral Code](https://m.do.co/c/2d2f25b376ae)!*   
*Need a domain? Get $5 off your first one through my [Name.com Referral Code](https://www.name.com/referral/302312)!*</sup>   
Assuming you have everything above squared away you can continue!

## Definitions

### Matrix
{:.no_toc}
[Matrix](https://en.wikipedia.org/wiki/Matrix_(protocol)) is a Real-Time Communication Protocol and Open Standard that relies on the federation of "homeservers" to build a communication network.

### Matrix.org
{:.no_toc}
[Matrix.org](https://matrix.org/foundation/) is the non-profit foundation behind the Matrix protocol. They also develop and distribute Synapse.

### Element
{:.no_toc}
[Element](https://en.wikipedia.org/wiki/Element_(software)) is both a Matrix Client and the name of the company behind said Client

### Synapse
{:.no_toc}
[Synapse](https://matrix.org/docs/projects/server/synapse) is the most popular homeserver implementation. Created by the Matrix.org team and the subject of this blog post.

### Homeserver
{:.no_toc}
A homeserver is the term used by Matrix to specify that it's a Matrix server and not a web server, a virtual private server, a minecraft server, etc. It is the "home" of the users registered at your server and where the messages from the channels of other homeservers they have joined will be collected.

### Federation
{:.no_toc}
Federation is the inter-connectivity of independent and distinct systems using a formalized protocol. Anyone can set-up their own Matrix homeserver and connect it to a federated Matrix network.

## Instructions

### Domain Preparation
As one of the first steps of setting up Synapse involves informing it of the domain being used, it makes sense to talk about it first.

With Matrix, the domain, aka "server name", is used as an identifier for the homesever and local users when interacting with other servers and users. Changing the domain will be treated as making a completely new homeserver. There is [a proposal](https://github.com/matrix-org/matrix-doc/blob/neilalexander/identities/proposals/2787-portable-identities.md) for Portable Identities which might help with server migration but as of now it's still in drafts, so choose carefully now to avoid potential headaches. 

A homeserver's server name plays a key factor in the identifier for the users and rooms created on it, providing a namespace to reduce collisions between names on ever growing networks. It also can function as quick way to distinguish your association, e.g. `@jake:statefarm.com`, `@bear:big.blue.house`, or `@geralt:riv.ia`. Given that, it's recommended that you have your homeserver use a clean version of your domain; `@neo:zion.city` would be a better choice than `@neo:chat.zion.city`, for example.

With all that out of the way, let's begin planning.

Both Clients and other Homeservers will attempt to connect with yours using HTTPS, however they both default to using different ports. Client-Server communication defaults to standard `443` of HTTPS while Server-Server communication uses `8448`. Synapse, of course, listens for both clients and other servers over HTTP on port `8008`; that is where using a reverse proxy, in our case traefik, comes in, which we will configure later.

If assigning port `443` (or `8448`) on your domain is an issue, there are alternative methods avaliable to let you keep it as your server name. These include configuring your reverse proxy to forward only [specific paths/locations to Matrix](https://matrix-org.github.io/synapse/latest/reverse_proxy.html) or add forwarding info to another domain, either via [serving a JSON file](https://matrix-org.github.io/synapse/latest/delegate.html#well-known-delegation) or [using a SRV record](https://matrix-org.github.io/synapse/latest/delegate.html#srv-dns-record-delegation). Neither will be covered in this post directly.

Assuming you don't have those issues, the only real step here is to make sure your domain is correctly pointed towards your server!

### Generating a Configuration File

In the working directory of your choice, create the following as `docker-compose.yml`

```yml
version: "3.7"

services:
  synapse:
    image: "matrixdotorg/synapse:latest"
    container_name: "yournamingscheme-synapse"
    environment:
      SYNAPSE_SERVER_NAME: "example.org"
      SYNAPSE_REPORT_STATS: "yes"
    volumes:
      - type: "bind"
        source: "${PWD}/synapse"
        target: "/data"
    restart: "unless-stopped"
    ulimits:
      nofile:
        soft: 131072
        hard: 131072
```
In your instance:
- replace `yournamingscheme-synapse` with a naming scheme of your choice.   
- replace `example.org` with your server name of choice
- optionally, change `SYNAPSE_REPORT_STATS` to `"no"` if you don't want to share information with Matrix.org
- optionally, change the `source` of the volume to a different location.

The `ulimits` setting extends the number of file handlers used within the container, the default amount can cause issues for the server

then create a `synapse` directory in your working directory

once ready run `docker-compose run synapse generate` and you should get output like this:
```shell
julian@server:/docker/yournamingscheme/synapse$ docker-compose run synapse generate
Creating log config /data/example.org.log.config
Generating config file /data/homeserver.yaml
Generating signing key file /data/example.org.signing.key
A config file has been generated in '/data/homeserver.yaml' for server name 'example.org'. Please review this file and customise it to your needs.
```
It may seem unresponsive for a short while during the generation but it should complete in a reasonable timeframe.

### Editing 'homeserver.yaml'

Using nano, or your text editor of choice, open the homeserver.yaml: `sudo nano ./synapse/homeserver.yaml`.   
There are a lot of configuration options, most with quick summaries commented above them. For now though we will focus on a select few, though I do recommend always reading the documentation.

#### \#\# Server \#\#
{:.no_toc}
From the top down you should see `server_name`, this should already be filled with the value you passed into the container via `SYNAPSE_SERVER_NAME`.

Next is the listener section, the only change to make is to remove `bind_addresses: ['::1', '127.0.0.1']` from the configuration for port 8008. Synapse will be using HTTP since Traefik will be unwrapping the HTTPS for it.

#### \#\# Database \#\#
{:.no_toc} 
By default Synapse generates and uses a SQite3 database, however we will be switching it to Postgres so it can handle an average load.

Replace the contents of the database section with the following:
```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: strongGeneratedPasswordHere
    host: yournamingscheme-postgres
    cp_min: 5
    cp_max: 10
```
Remember the configuration added here as it will be reused in the `docker-compose.yml` when adding the database

#### \#\# Registration \#\#
{:.no_toc}
Assuming you want users to be able to join your home server, you should enable registeration with `enable_registration: true`. In this section you can also configure what the user needs when registering, e.g., email, phone number, password requirements. Synapse and Matrix are very requirement agnostic, letting homeserver owners tune it to their specifications.

Do note that for items like email and single-sign on, further configuration will be needed in lower sections.

### Adding Postgres to docker-compose.yml

Update your `docker-compose.yml` with configuration for postgres:
```yaml
version: "3.7"

services:
  synapse:
    image: "matrixdotorg/synapse:latest"
    container_name: "yournamingscheme-synapse"
    environment:
      SYNAPSE_SERVER_NAME: "example.org"
      SYNAPSE_REPORT_STATS: "yes"
    volumes:
      - type: "bind"
        source: "${PWD}/synapse"
        target: "/data"
    restart: "unless-stopped"
    ulimits:
      nofile:
        soft: 131072
        hard: 131072
    depends_on:
      - "postgres"
  postgres:
    image: "postgres:14"
    container_name: "yournamingscheme-postgres"
    environment:
      POSTGRES_USER: "synapse"
      POSTGRES_PASSWORD: "useThePasswordYouPutInHomeserverYAML"
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    volumes:
      - type: "bind"
        source: "${PWD}/postgres"
        target: "/var/lib/postgresql/data"
    restart: "unless-stopped"
```
`container_name` and `POSTGRES_PASSWORD` should match the values for `host` and `user` you used in `homesever.yaml` within the database section.

Then create a `postgres` directory in your working directory

### Create your first user

To create your first user, we'll be using the `register_new_matrix_user`. However since we are using docker it will take a few extra steps.

First start your services with `docker-compose up -d`.   
Then run `docker-compose exec synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008`.   
This will bring up input prompts asking for a "New user localpart" (username), password, and if the user should be an admin (yes).

Once completed, all that's left is to configure traefik to allow access to your homeserver

### Traefik configuration

If you running Traefik from a Docker Container, make sure the ports used `443` and `8448` are exposed. As well, we will need to configure entrypoints for our routers to use. If you already have a HTTPS route on `443` you can reuse that otherwise add the following to your `traefik.toml` under `[entryPoints]`
```toml
  [entryPoints.webSecure]
		address = ":443"
		http.tls.certResolver = "lets-encrypt"
	[entryPoints.matrixSecure]
		address = ":8448"
		http.tls.certResolver = "lets-encrypt"
```

Make sure to stop and restart your Traefik instance for these changes to take effect.

Next, update your `docker-compose.yml` to the following:
```yml
version: "3.7"

services:
  synapse:
    image: "matrixdotorg/synapse:latest"
    container_name: "yournamingscheme-synapse"
    environment:
      SYNAPSE_SERVER_NAME: "example.org"
      SYNAPSE_REPORT_STATS: "yes"
    volumes:
      - type: "bind"
        source: "${PWD}/synapse"
        target: "/data"
    restart: "unless-stopped"
    ulimits:
      nofile:
        soft: 131072
        hard: 131072
    depends_on:
      - "postgres"
    labels:
      - "traefik.http.services.synapse.loadbalancer.server.port=8008"
      - "traefik.http.routers.synapseServer.rule=Host(`example.org`)"
      - "traefik.http.routers.synapseServer.entrypoints=matrixSecure"
      - "traefik.http.routers.synapseServer.service=synapse"
      - "traefik.http.routers.synapseClient.rule=Host(`example.org`)"
      - "traefik.http.routers.synapseClient.entrypoints=webSecure"
      - "traefik.http.routers.synapseClient.service=synapse"
  postgres:
    image: "postgres:14"
    container_name: "yournamingscheme-postgres"
    environment:
      POSTGRES_USER: "synapse"
      POSTGRES_PASSWORD: "useThePasswordYouPutInHomeserverYAML"
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    volumes:
      - type: "bind"
        source: "${PWD}/postgres"
        target: "/var/lib/postgresql/data"
    restart: "unless-stopped"
    labels:
      - "traefik.enable=false"
```

Make sure to stop and restart your Traefik instance for these changes to take effect.

If you need Synapse to share the webSecure port on the host, make sure to read up on Delegation as mentioned in the Domain section.   

Once the services are back up, check the Traefik dashboard to see if they are configured. You can also use the [Federation Tester](https://federationtester.matrix.org/) to see if your Synapse server is accessible to other homeservers.

All that's left is to get a Matrix client, point it at your homeserver, and log in!

## Conclusion

Congrats, assuming things all worked correctly, you have a working Synapse server!

Ways to go from here include: setting up a TURN server, further configuration of the user authenication, and reading the documentation.

Hopefully this has helped!

## Further Reading
* [Synapse Documentation](https://matrix-org.github.io/synapse/latest/welcome_and_overview.html)
* [Matrix Specification](https://spec.matrix.org/v1.1/)