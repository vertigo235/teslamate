---
title: Advanced install with Traefik, Let's Encrypt & HTTP Basic Auth
---

In case you wish to make TeslaMate publicly available on the Internet, it is strongly recommended to secure the web interface and allow access to Grafana only with a password. This guide provides a _[docker-compose.yml](#docker-composeyml)_ which differs from the basic installation in the following aspects:

- Both publicly accessible services, TeslaMate and Grafana, sit behind a reverse proxy (Traefik) which terminates HTTPS traffic
- The TeslaMate service is protected by HTTP Basic Authentication
- Custom configuration was moved into a separate `.env` file
- A Let's Encrypt certificate is acquired automatically
- Grafana is configured to require a login

:::note
Please note that this is only **an example** of how TeslaMate can be used in a more advanced scenario. Depending on your use case, you may need to make some adjustments, primarily to the traefik configuration. For more information, see the [traefik docs](https://docs.traefik.io/).
:::

## Requirements

- Two FQDN, for example `teslamate.example.com` and `grafana.example.com`

## Instructions

Create the following three files:

### docker-compose.yml

```yml title="docker-compose.yml"
version: "3"

services:
  teslamate:
    image: teslamate/teslamate:latest
    restart: always
    depends_on:
      - database
    environment:
      - DATABASE_USER=${TM_DB_USER}
      - DATABASE_PASS=${TM_DB_PASS}
      - DATABASE_NAME=${TM_DB_NAME}
      - DATABASE_HOST=database
      - MQTT_HOST=mosquitto
      - VIRTUAL_HOST=${FQDN_TM}
      - CHECK_ORIGIN=true
      - TZ=${TM_TZ}
    volumes:
      - ./import:/opt/app/import
    labels:
      - "traefik.enable=true"
      - "traefik.port=4000"
      - "traefik.http.middlewares.redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.auth.basicauth.usersfile=/auth/.htpasswd"
      - "traefik.http.routers.teslamate-insecure.rule=Host(`${FQDN_TM}`)"
      - "traefik.http.routers.teslamate-insecure.middlewares=redirect"
      - "traefik.http.routers.teslamate-ws.rule=Host(`${FQDN_TM}`) && Path(`/live/websocket`)"
      - "traefik.http.routers.teslamate-ws.entrypoints=websecure"
      - "traefik.http.routers.teslamate-ws.tls"
      - "traefik.http.routers.teslamate.rule=Host(`${FQDN_TM}`)"
      - "traefik.http.routers.teslamate.middlewares=auth"
      - "traefik.http.routers.teslamate.entrypoints=websecure"
      - "traefik.http.routers.teslamate.tls.certresolver=tmhttpchallenge"
    cap_drop:
      - all

  database:
    image: postgres:12
    restart: always
    environment:
      - POSTGRES_USER=${TM_DB_USER}
      - POSTGRES_PASSWORD=${TM_DB_PASS}
      - POSTGRES_DB=${TM_DB_NAME}
    volumes:
      - teslamate-db:/var/lib/postgresql/data

  grafana:
    image: teslamate/grafana:latest
    restart: always
    environment:
      - DATABASE_USER=${TM_DB_USER}
      - DATABASE_PASS=${TM_DB_PASS}
      - DATABASE_NAME=${TM_DB_NAME}
      - DATABASE_HOST=database
      - GRAFANA_PASSWD=${GRAFANA_PW}
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PW}
      - GF_AUTH_BASIC_ENABLED=true
      - GF_AUTH_ANONYMOUS_ENABLED=false
      - GF_SERVER_ROOT_URL=https://${FQDN_GRAFANA}
    volumes:
      - teslamate-grafana-data:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.port=3000"
      - "traefik.http.middlewares.redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.grafana-insecure.rule=Host(`${FQDN_GRAFANA}`)"
      - "traefik.http.routers.grafana-insecure.middlewares=redirect"
      - "traefik.http.routers.grafana.rule=Host(`${FQDN_GRAFANA}`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls.certresolver=tmhttpchallenge"

  mosquitto:
    image: eclipse-mosquitto:1.6
    restart: always
    ports:
      - 127.0.0.1:1883:1883
    volumes:
      - mosquitto-conf:/mosquitto/config
      - mosquitto-data:/mosquitto/data

  proxy:
    image: traefik:v2.2
    restart: always
    command:
      - "--global.sendAnonymousUsage=false"
      - "--providers.docker"
      - "--providers.docker.exposedByDefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.tmhttpchallenge.acme.httpchallenge=true"
      - "--certificatesresolvers.tmhttpchallenge.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.tmhttpchallenge.acme.email=${LETSENCRYPT_EMAIL}"
      - "--certificatesresolvers.tmhttpchallenge.acme.storage=/etc/acme/acme.json"
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./.htpasswd:/auth/.htpasswd
      - ./acme/:/etc/acme/
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  teslamate-db:
  teslamate-grafana-data:
  mosquitto-conf:
  mosquitto-data:
```

:::note
If you are upgrading from the [simple Docker setup](../installation/docker.md) make sure that you are using the same Postgres version as before. To upgrade to a new version see [Upgrading PostgreSQL](../maintenance/upgrading_postgres.md).
:::

### .env

```plaintext title=".env"
TM_DB_USER=teslamate
TM_DB_PASS=secret
TM_DB_NAME=teslamate

GRAFANA_USER=admin
GRAFANA_PW=admin

FQDN_GRAFANA=grafana.example.com
FQDN_TM=teslamate.example.com

TM_TZ=Europe/Berlin

LETSENCRYPT_EMAIL=yourperson@example.com
```

:::note
If you are upgrading from the [simple Docker setup](../installation/docker.md) make sure to use the same database and Grafana credentials as before.
:::

### .htpasswd

This file contains a user and password for accessing TeslaMate (Basic-auth), note this is NOT your tesla.com password. You can generate it on the web if you don't have the [Apache tools](https://www.cyberciti.biz/faq/create-update-user-authentication-files/) installed (e.g. http://www.htaccesstools.com/htpasswd-generator/).

**Example:**

```apacheconf title=".htpasswd"
teslamate:$apr1$0hau3aWq$yzNEh.ABwZBAIEYZ6WfbH/
```

## Usage

Start the stack with `docker-compose up`.

1. Open the web interface [tesla.example.com](https://tesla.example.com)
2. Sign in with your Tesla account
3. The Grafana dashboards are available at [grafana.example.com](https://grafana.example.com).

:::tip
If you have difficulty logging into your Grafana i.e. you cannot login with the credentials from either the simple setup or the values stored in the .env file reset the admin password with the following command:

```
docker-compose exec grafana grafana-cli admin reset-admin-password
```

:::
