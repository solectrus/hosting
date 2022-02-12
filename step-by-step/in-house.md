# Install Solectrus on an in-house server

This example demonstrates how to run **all** components of Solectrus on a single machine:

- Dashboard (the Solectrus UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database, not used yet)
- SENEC-Collector (live-pulling measurements from SENEC devise and pushing them the InfluxDB)
- Forecast-Collector (optional)
- Renault-Collector (optional)

This example is tested on a Synology NAS, but should work on any Linux system with `x86_64` architecture which can run Docker.

This machine connects to your SENEC device, so it should be placed in the same network.

1. Ensure Docker is installed and running. Docker 18.06.0 or later is required.

```
$ docker --version
Docker version 20.10.3, build b455053
```

2. Create folders for configuration and Docker volumes:

On a Synology NAS, there is a `/volume1/docker` folder which is used for Docker volumes. This may be different on your machine. Select a folder of your choice.

```
cd /volume1/docker
mkdir solectrus
cd solectrus
mkdir redis
mkdir postgresql
mkdir influxdb
```

3. Create configuration file

Create a file named `/volume1/docker/solectrus/.env` and add the following lines (change the values to your needs):

```
##################################################################
# Dashboard application (the main part)
#
# Domain name or IP address of your host
APP_HOST=name-of-your-server
#
# SSL redirect: Use "true" (which is the default) if you want to auto-redirect to https
FORCE_SSL=false
#
# Secret token to secure cookies, 128 chars long hexadecimal encoded string (don't use this example, make some random changes!)
# Currently there are no cookies in Soletrus, but this may change in the future
SECRET_KEY_BASE=f60debe97dcb73280a2cc83668fd60e8d0e8e48269036a7bce980ee53cfb312e377989a750b8c945a5f69b041289ecb4e2d9e40641b81257c65ac2d43e3c837f
#
# Date of commissioning of your photovoltaic system
INSTALLATION_DATE=2022-01-27
#
# Price you pay for 1kWH (in EUR)
ELECTRICITY_PRICE=0.28
#
# Price you get for 1kWH (in EUR)
FEED_IN_TARIFF=0.0848
#
# Password for the PostgreSQL database (currently not used)
POSTGRES_PASSWORD=db-password


##################################################################
# Influx database settings
#
# Influx host (to access from SOLECTRUS dashboard and collectors)
INFLUX_HOST=influxdb
INFLUX_SCHEMA=http
INFLUX_PORT=8086
#
# Credentials for the Influx database
INFLUX_ORG=solectrus
INFLUX_USERNAME=admin
INFLUX_PASSWORD=ExAmPl3PA55W0rD
INFLUX_ADMIN_TOKEN=my-super-secret-admin-token
INFLUX_BUCKET=my-solectrus-bucket
#
# To keep things simple, we use ONE token (INFLUX_ADMIN_TOKEN) for both writing and reading.
# For better security, it is recommended NOT to use the admin token here. Instead,
# create two separate tokens for writing and reading (via the InfluxDB frontend).
INFLUX_TOKEN_WRITE=my-super-secret-admin-token
INFLUX_TOKEN_READ=my-super-secret-admin-token
#
# Volume path for storing the Influx data
INFLUX_VOLUME_PATH=/volume1/docker/solectrus/dashboard/influxdb


##################################################################
# REDIS settings
#
REDIS_VOLUME_PATH=/volume1/docker/solectrus/dashboard/redis


##################################################################
# PostgreSQL database settings
#
DB_VOLUME_PATH=/volume1/docker/solectrus/dashboard/postgresql


##################################################################
# SENEC-Collector
#
SENEC_HOST=192.168.xxx.xxx # change this!!!
SENEC_INTERVAL=5

##################################################################
# Solar forecasting with https://forecast.solar
# API docs: https://doc.forecast.solar/doku.php?id=api:estimate
#
# Latitude of the plant location
FORECAST_LATITUDE=50.92149
#
# Longitude of the plant location
FORECAST_LONGITUDE=6.36267
#
# Plane declination: 0 (horizontal) - 90 (vertical)
FORECAST_DECLINATION=30
#
# Plane azimuth: -180 ... 180 (-180 = north, -90 = east, 0 = south, 90 = west, 180 = north)
FORECAST_AZIMUTH=20
#
# Installed modules power in kilowatt peak (kWp)
FORECAST_KWP=8.4
#
# Update interval in seconds, 900s = 15 minutes, the public (and free) API allows a minimum of 900 seconds
FORECAST_INTERVAL=900
```

3. Create Docker compose file `/volume1/docker/solectrus/docker-compose.yml`

```
version: "3.7"

services:
  app:
    image: ghcr.io/solectrus/solectrus:latest
    depends_on:
      - db
      - influxdb
      - redis
    links:
      - db
      - influxdb
      - redis
    ports:
      - 3000:3000
    environment:
      - APP_HOST
      - FORCE_SSL
      - SECRET_KEY_BASE
      - INSTALLATION_DATE
      - ELECTRICITY_PRICE
      - FEED_IN_TARIFF
      - DB_HOST=db
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - DB_USER=postgres
      - REDIS_URL=redis://redis:6379/1
      - INFLUX_HOST
      - INFLUX_TOKEN=${INFLUX_TOKEN_READ}
      - INFLUX_ORG
      - INFLUX_BUCKET
    healthcheck:
      test: ["CMD-SHELL", "nc -z 127.0.0.1 3000 || exit 1"]
    restart: always

  influxdb:
    image: influxdb:2.1.1-alpine
    volumes:
      - ${INFLUX_VOLUME_PATH}:/var/lib/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=${INFLUX_USERNAME}
      - DOCKER_INFLUXDB_INIT_PASSWORD=${INFLUX_PASSWORD}
      - DOCKER_INFLUXDB_INIT_ORG=${INFLUX_ORG}
      - DOCKER_INFLUXDB_INIT_BUCKET=${INFLUX_BUCKET}
      - DOCKER_INFLUXDB_INIT_ADMIN_TOKEN=${INFLUX_ADMIN_TOKEN}
    command: influxd run --bolt-path /var/lib/influxdb2/influxd.bolt --engine-path /var/lib/influxdb2/engine --store disk
    restart: always

  db:
    image: postgres:14-alpine
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - ${DB_VOLUME_PATH}:/var/lib/postgresql/data
    restart: always

  redis:
    image: redis:alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
    volumes:
      - ${REDIS_VOLUME_PATH}:/data
    restart: always

  senec-collector:
    image: ghcr.io/solectrus/senec-collector:latest
    depends_on:
      - influxdb
    links:
      - influxdb
    environment:
      - SENEC_HOST
      - SENEC_INTERVAL
      - INFLUX_HOST
      - INFLUX_SCHEMA
      - INFLUX_PORT
      - INFLUX_TOKEN=${INFLUX_TOKEN_WRITE}
      - INFLUX_ORG
      - INFLUX_BUCKET
    restart: always

  #renault-collector:
  #  image: ghcr.io/solectrus/renault-collector:latest
  #  links:
  #    - influxdb
  #  depends_on:
  #    - influxdb
  #  environment:
  #    - RENAULT_EMAIL
  #    - RENAULT_PASSWORD
  #    - RENAULT_INTERVAL
  #    - RENAULT_MODEL
  #    - GIGYA_API_KEY
  #    - KAMEREON_API_KEY
  #    - INFLUX_HOST
  #    - INFLUX_TOKEN=${INFLUX_TOKEN_WRITE}
  #    - INFLUX_ORG
  #    - INFLUX_BUCKET
  #  command: bundle exec app/main.rb
  #  restart: always

  #forecast-collector:
  #  image: ghcr.io/solectrus/forecast-collector:latest
  #  depends_on:
  #    - influxdb
  #  links:
  #    - influxdb
  #  environment:
  #    - INFLUX_HOST
  #    - INFLUX_TOKEN=${INFLUX_TOKEN_WRITE}
  #    - INFLUX_ORG
  #    - INFLUX_BUCKET
  #    - FORECAST_LATITUDE
  #    - FORECAST_LONGITUDE
  #    - FORECAST_DECLINATION
  #    - FORECAST_AZIMUTH
  #    - FORECAST_KWP
  #    - FORECAST_INTERVAL
  #  restart: always
```

4. Start Docker containers

```
docker-compose up
```

This could take some minutes for the first run. You see some output like this:

```
.... lots of other lines here ....
app_1     | Created database 'solectrus_production'
app_1     | Database is ready!
app_1     | Puma starting in single mode...
app_1     | * Puma version: 5.6.2 (ruby 3.1.0-p0) ("Birdie's Version")
app_1     | *  Min threads: 5
app_1     | *  Max threads: 5
app_1     | *  Environment: production
app_1     | *          PID: 76
app_1     | * Listening on http://0.0.0.0:3000
app_1     | Use Ctrl-C to stop
```

5. Open the app in your browser

http://name-of-your-server:3000
