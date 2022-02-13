# Install Solectrus on an in-house server

This example demonstrates how to run **all** components of Solectrus on a single machine:

- Dashboard (the Solectrus UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database, not used yet)
- SENEC Collector (live-pulling measurements from SENEC devise and pushing them the InfluxDB)
- Forecast-Collector (optional)
- Renault-Collector (optional)

This example is tested on a Synology NAS, but should work on any Linux system with `x86_64` architecture which can run Docker.

This machine connects to your SENEC device, so it should be placed in the same network.

1. Ensure Docker and Docker Compose is installed and running. Docker 18.06.0 or later is required.

```
$ docker --version
Docker version 20.10.3, build b455053

$ docker-compose -v
docker-compose version 1.28.5, build 324b023a
```

2. Create folders for configuration and Docker volumes:

On a Synology NAS, there is a `/volume1/docker` folder which is used for Docker volumes. This may be different on your machine.

```
cd /volume1/docker
mkdir solectrus
cd solectrus
mkdir redis
mkdir postgresql
mkdir influxdb
```

3. Download and prepare configuration file

Download the configuration file from the repository;

```
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/in-house/.env" -o /volume1/docker/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```
$ vim .env
```

You should change at least the following values:

- INSTALLATION_DATE
- ELECTRICITY_PRICE
- FEED_IN_TARIFF
- FORECAST_LATITUDE
- FORECAST_LONGITUDE
- FORECAST_DECLINATION
- FORECAST_AZIMUTH
- FORECAST_KWP


4. Download Docker compose file `./docker-compose.yml`

```
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/in-house/docker-compose.yml" -o /volume1/docker/solectrus/docker-compose.yml
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

http://[[YOUR-SERVER-IP-ADDRESS]]:3000


6. Run services in the background

Stop services by pressing ^C
Start again as daemon:

```
$ docker-compose up -d
Starting solectrus_redis_1    ... done
Starting solectrus_db_1       ... done
Starting solectrus_influxdb_1 ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_app_1                ... done
```

This way the services are started in the background. To check if this works, reboot machine:

```
$ sudo reboot
```

Wait a bit, then login again and check if Docker Compose has auto-started the services.

```
$ ssh root@[YOUR-SERVER-IP-ADDRESS]
$ cd solectrus
$ docker ps
NAMES                            STATUS
solectrus_app_1                  Up 31 seconds (healthy)
solectrus_forecast-collector_1   Up 31 seconds
solectrus_influxdb_1             Up 33 seconds
solectrus_redis_1                Up 33 seconds
solectrus_db_1                   Up 33 seconds
```
