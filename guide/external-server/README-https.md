# Securing your SOLECTRUS installation

This guide assumes that you have a working SOLECTURS installation that is reachable via http, e.g. after following the instructions for the [Distributed installation](README.md) or the [Remote installation with cloud access](../external-server-cloud/README.md).

To secure your SOLECTRUS installation using [traefik](https://github.com/traefik/traefik) and [Let's Encrypt](https://letsencrypt.com), follow these steps.

### a) make sure you have a DNS entry for your server

For this step, it is required that you own a domain and have access to its DNS records. How this is done in practice varies from hosting provider to hosting provider, but in general you should have access to a DNS or subdomain feature for your domain, where you can add a hostname for your newly configured SOLECTRUS server.

Using these tools, you need to choose a name for your subdomain and add the IP address of your SOLECTRUS server to a so-called "A" record.

For example, let's say you own `mydomain.de`, your server has IP address 1.2.3.4 and your chosen SOLECTURS subdomain is `solectrus.mydomain.de`. You will need to add an "A" record to your DNS that maps `solectrus.mydomain.de` to the address 1.2.3.4

To verify if this step was successful, try accessing `http://solectrus.mydomain.de` from your browser or use tools like `dig` or `ping` to check if the name resolution works.

### b) change the firewall configuration for your server

It is generally a good practice to protect your server using a firewall, and all hosting providers offer a way to do so. 

Go to your server's configuration console. If you do not already have a firewall configured, add one! This firewall must allow incoming TCP traffic on ports 22 (for ssh), 80 (for http), 443 (for https) and  8086 (for influxdb).

### c) add traefik and Let's Encrypt to your docker configuration

Log in to your SOLECTRUS server and go to the solectrus directory (`cd solectrus`). Now, perform these commands:

```
mkdir letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

Open your `docker-compose.yml` file in an editor and add this snippet:

```
traefik:
  image: "traefik:v2.11" 
  command:
    - "--providers.docker=true"
    - "--providers.docker.exposedbydefault=false"
    - "--entrypoints.web.address=:80"
    - "--entrypoints.websecure.address=:443"
    - "--entrypoints.influxdb.address=:8086"
    - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
    - "--certificatesresolvers.myresolver.acme.email=email@mydomain.de"
    - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
  ports:
    - "80:80"
    - "443:443"
    - "8086:8086"
  volumes:
    - "./letsencrypt:/letsencrypt"
    - "/var/run/docker.sock:/var/run/docker.sock:ro"
```

to the `services` section. Replace `email@mydomain.de` with an email address that you own. This adds the `traefik` proxy which will listen on ports 80 (for http), 443 (for https) and 8086 (for the influxdb collector).

### d) reconfigure the app 

Now, add this snippet to the `app` section of your docker-compose:

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.app-solectrus.rule=Host(`solectrus.mydomain.de`)"
  - "traefik.http.routers.app-solectrus.entrypoints=websecure"
  - "traefik.http.routers.app-solectrus.tls.certresolver=myresolver"
  - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
  - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
  - "traefik.http.routers.redirs.entrypoints=web"
  - "traefik.http.routers.redirs.middlewares=redirect-to-https"
```

(replacing `solectrus.mydomain.de` with your chosen subdomain name). This puts SOLECTRUS under traefik's control and allows access via `https://solectrus.mydomain.de`. Also, all requests to `http://solectrus.mydomain.de` will be redirected to `https://solectrus.mydomain.de`.

Also in the `app` section, remove these two lines:

```
ports:
  - 80:3000
```

### e) reconfigure influxdb

Then, add this snippet to the `influxdb` section:

```
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.influxdb-solectrus.rule=Host(`solectrus.mydomain.de`)"
  - "traefik.http.routers.influxdb-solectrus.entrypoints=influxdb"
  - "traefik.http.routers.influxdb-solectrus.tls.certresolver=myresolver"
  - "traefik.http.routers.influxdb-solectrus.tls=true"
```

Again, replacing `solectrus.mydomain.de` with your chosen subdomain. This puts the InfluxDB database under traefik's control and makes it avaiable via `https://solectrus.mydomain.de:8086`.

Similar to the `app` change, remove these two lines:

```
ports:
  - 8086:8086
```

Save your changes to docker-compose.yml.

Your file should now look like this:

```
version: '3.7'

services:
  traefik:
    image: "traefik:v2.11"
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.influxdb.address=:8086"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=email@mydomain.de"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
      - "8086:8086"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

  app:
    image: ghcr.io/solectrus/solectrus:develop
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app-solectrus.rule=Host(`solectrus.mydomain.de`)"
      - "traefik.http.routers.app-solectrus.entrypoints=websecure"
      - "traefik.http.routers.app-solectrus.tls.certresolver=myresolver"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"
      - "traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)"
      - "traefik.http.routers.redirs.entrypoints=web"
      - "traefik.http.routers.redirs.middlewares=redirect-to-https"
    depends_on:
      ... original configuration here, without "ports" ...
    restart: always

  influxdb:
    image: influxdb:2.7-alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.influxdb-solectrus.rule=Host(`solectrus.mydomain.de`)"
      - "traefik.http.routers.influxdb-solectrus.entrypoints=influxdb"
      - "traefik.http.routers.influxdb-solectrus.tls.certresolver=myresolver"
      - "traefik.http.routers.influxdb-solectrus.tls=true"
    volumes:
      ... original configuration here, without "ports" ...
      start_period: 30s
      
   ... more original configuration here ...
```

You should now be able to restart your containers using `docker compose up -d`. After all containers have been restarted, traefik will automatically fetch (and later renew) a TLS certificate for your subdomain, this process will take a few minutes at most. After waiting a little, you should be able to access `https://solectrus.mydomain.de` without getting any security warnings in your browser. `https://solectrus.mydomain.de:8086` should should the InfluxDB admin login page.

Note that whenever you restart traefik or the solectrus-app container, it takes a short while for traefik to be fully available again. Getting "404 page not found" responses immediately after restarting is normal and no cause for concern.

### f) (optional) add the host name and SSL support to your .env

Optionally, you can edit your `.env` file and change the settings for `APP_HOST` and `FORCE_SSL`
The new values should be

```
APP_HOST=solectrus.mydomain.de # replace with your subdomain name
FORCE_SSL=true
```

This will enable redirecting requests to `http://[YOUR-SERVER-IP-ADDRESS]` to `https://solectrus.mydomain.de`. (You will have to 

### g) reconfigure your senec-collector to use the encrypted connection

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
         -e INFLUX_HOST=solectrus.mydomain.de \
         -e INFLUX_SCHEMA=https \
         -e INFLUX_ORG=solectrus \
         -e INFLUX_BUCKET=solectrus \
         -e INFLUX_TOKEN=my-super-secret-admin-token \
         ghcr.io/solectrus/senec-collector:latest
```

Again, replacing `solectrus.mydomain.de` with your chosen subdomain and adding `INFLUX_SCHEMA` to use https to talk to your influxdb. 

Restart senec-collector, you should now see new measurement data on `https://solectrus.mydomain.de`.

Congratulations! :-)