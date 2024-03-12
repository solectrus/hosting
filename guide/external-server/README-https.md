# Securing your SOLECTRUS installation

This guide assumes that you have a working SOLECTURS installation that is reachable via http, e.g. after following the instructions for the [Distributed installation](README.md) or the [Remote installation with cloud access](../external-server-cloud/README.md).

To secure your SOLECTRUS installation using [traefik](https://github.com/traefik/traefik) and [Let's Encrypt](https://letsencrypt.com), follow these steps.

### a) make sure you have a DNS entry for your server

For this step, it is required that you own a domain and have access to its DNS records. How this is done in practice varies from hosting provider to hosting provider, but in general you should have access to a DNS or subdomain feature for your domain, where you can add a hostname for your newly configured SOLECTRUS server.

Using these tools, you need to choose a name for your subdomain and add the IP address of your SOLECTRUS server to a so-called "A" record.

For example, let's say you own `mydomain.de`, your server has IP address 1.2.3.4 and your chosen SOLECTURS subdomain is `solectrus.mydomain.de`. You will need to add an "A" record to your DNS that maps `solectrus.mydomain.de` to the address 1.2.3.4

To verify if this step was successful, try accessing `http://solectrus.mydomain.de` from your browser.

### b) change the firewall configuration for your server

Go to your server configuration console. Add a firewall rule that allows incoming traffic on port 8186. (Ports 22, 80 and 443 should remain as allowed). 

Any previous rule allowing incoming traffic on port 8086 can be removed as it is no longer needed.

### c) add traefik and Let's Encrypt to your docker configuration

Log in to your SOLECTRUS server and go to the solectrus directory (`cd solectrus`). Now, perform these commands:

```
mkdir letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

Open your docker-compose.yml file in an editor and add this snippet:

```
  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.influxdb.address=:8186"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=EMAIL@MYDOMAIN.DE"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8186:8186"
      - "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

```

to the `services` section. Replace `EMAIL@MYDOMAIN.DE` with an email address that you own. This adds the `traefik` proxy which will listen on ports 80 (for `http`), 443 (for `https`) and 8186 (for the influxdb collector).

Now, add this snippet to the `app` section of your docker-compose:

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.app-solectrus.rule=Host(`SOLECTRUS.MYDOMAIN.DE`)"
  - "traefik.http.routers.app-solectrus.entrypoints=websecure"
  - "traefik.http.routers.app-solectrus.tls.certresolver=myresolver"
  - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
  - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
  - "traefik.http.routers.redirs.entrypoints=web"
  - "traefik.http.routers.redirs.middlewares=redirect-to-https"
```

(replacing `SOLECTRUS.MYDOMAIN.DE` with your chosen subdomain name). This puts SOLECTRUS under traefik's control and allows access via `https://solectrus.mydomain.de`. Also, all requests to `http://solectrus.mydomain.de` will be redirected to `https://solectrus.mydomain.de`.

Lastly, add this snippet to the `influxdb` section:

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.influxdb-solectrus.rule=Host(`SOLECTRUS.MYDOMAIN.DE`)"
  - "traefik.http.routers.influxdb-solectrus.entrypoints=influxdb"
  - "traefik.http.routers.influxdb-solectrus.tls.certresolver=myresolver"
  - "traefik.http.routers.influxdb-solectrus.tls=true"

```

Again, replacing `SOLECTRUS.MYDOMAIN.DE` with your chosen subdomain. This puts the InfluxDB database under traefik's control and makes it avaiable via `https://solectrus.mydomain.de:8186` 

Save your changes to docker-compose.yml.

Your file should now look like this:

```
version: '3.7'

services:
  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.influxdb.address=:8186"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=EMAIL@MYDOMAIN.DE"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8186:8186"
      - "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  app:
    image: ghcr.io/solectrus/solectrus:develop
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app-solectrus.rule=Host(`SOLECTRUS.MYDOMAIN.DE`)"
      - "traefik.http.routers.app-solectrus.entrypoints=websecure"
      - "traefik.http.routers.app-solectrus.tls.certresolver=myresolver"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.redirs.entrypoints=web"
      - "traefik.http.routers.redirs.middlewares=redirect-to-https"
    depends_on:
      ... original configuration here ...
    restart: always

  influxdb:
    image: influxdb:2.7-alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.influxdb-solectrus.rule=Host(`SOLECTRUS.MYDOMAIN.DE`)"
      - "traefik.http.routers.influxdb-solectrus.entrypoints=influxdb"
      - "traefik.http.routers.influxdb-solectrus.tls.certresolver=myresolver"
      - "traefik.http.routers.influxdb-solectrus.tls=true"
    volumes:
      ... original configuration here ...
      start_period: 30s
      
   ... more original configuration here ...
```

You should now be able to restart your containers using `docker compose up -d`. After all containers have been restarted, traefik will automatically fetch (and later renew) a TLS certificate for your subdomain, this process will take a few minutes at most. After waiting a little, you should be able to access `https://solectrus.mydomain.de` without getting any security warnings in your browser. `https://solectrus.mydomain.de:8186` should should the InfluxDB admin login page.

Note that whenever you restart the solectrus-app container, it takes a short while for traefik to notice that it is available again. Getting "404 page not found" responses immediately after restarting is normal and no cause for concern.

### d) reconfigure your senec-collector to use the encrypted connection

Login to your Raspberry Pi and change the startup command for your senec collector to the following:

```
docker run \
         --name senec-collector \
         --privileged \
         --restart unless-stopped \
         -d \
         -e SENEC_HOST=[YOUR-SENEC-IP-ADDRESS] \
         -e SENEC_SCHEMA=https \
         -e SENEC_INTERVAL=5 \
         -e SENEC_LANGUAGE=de \
         -e INFLUX_PORT=8186 \
         -e INFLUX_HOST=SOLECTRUS.MYDOMAIN.DE \
         -e INFLUX_PORT=8186 \
         -e INFLUX_SCHEMA=https \
         -e INFLUX_ORG=solectrus \
         -e INFLUX_BUCKET=solectrus \
         -e INFLUX_TOKEN=my-super-secret-admin-token \
         ghcr.io/solectrus/senec-collector:latest
```

Again, replacing SOLECTRUS.MYDOMAIN.DE with your chosen subdomain. This adds `INFLUX_PORT` and `INFLUX_SCHEMA` to use https on your newly defined port 8186 which uses `https`. Restart senec-collector, you should now see new measurement data on `https://solectrus.mydomain.de`.

Congratulations! :-)