# Install Solectrus on an in-house server

This example demonstrates how to run **all** components of Solectrus on a single machine:

- Dashboard (the Solectrus UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database for storing settings like historical prices)
- SENEC Collector (pulling measurements from SENEC devise and pushing them the InfluxDB)
- Forecast-Collector (optional)
- Renault-Collector (optional)

This example is tested on a Synology NAS DS220+, but should work on any Linux system with `x86_64` architecture which can run Docker. A CPU with at least 2 cores is recommended, as well as a RAM upgrade to more than 2GB. A Linux Kernel v4 or higher is **required**, some older NAS devices don't work because they are on Kernel v3 and cannot be updated.

This machine connects to your SENEC device, so it should be placed in the same network.

1. Ensure Docker and Docker Compose is installed and running. Docker 18.06.0 or later is required.

```console
docker --version
Docker version 20.10.3, build 55f0773

docker-compose -v
docker-compose version 1.28.5, build 24fb474e
```

2. Create folders for configuration and Docker volumes:

On a Synology NAS, there is a `/volume1/docker` folder which is used for Docker volumes. This may be different on your machine.

```console
ssh [YOUR-SERVER-IP-ADDRESS]
cd /volume1/docker
mkdir -p solectrus
cd solectrus
mkdir redis postgresql influxdb
```

3. Download and prepare configuration file

Download the configuration file from the repository;

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/in-house/.env" -o /volume1/docker/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```console
vim .env
```

You should change at least the following values:

- ADMIN_PASSWORD
- INSTALLATION_DATE
- ELECTRICITY_PRICE
- FEED_IN_TARIFF
- FORECAST_LATITUDE
- FORECAST_LONGITUDE
- FORECAST_DECLINATION
- FORECAST_AZIMUTH
- FORECAST_KWP

Note: Editing the file with `vim` is just an example, because this is the pre-installed editor. You can use any editor you like. If you are not familiar with `vim`, you can use [TextEditor](https://www.synology.com/dsm/packages/TextEditor) by Synology which can be installed as package via DSM.


4. Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/in-house/docker-compose.yml" -o /volume1/docker/solectrus/docker-compose.yml
```

5. Start Docker containers

```console
docker-compose up
```

This could take some minutes for the first run. You see some output like this:

```
.... lots of other lines here ....
app_1     | Created database 'solectrus_production'
app_1     | Database is ready!
app_1     | Puma starting in single mode...
app_1     | * Puma version: 5.6.5 (ruby 3.1.2-p20) ("Birdie's Version")
app_1     | *  Min threads: 5
app_1     | *  Max threads: 5
app_1     | *  Environment: production
app_1     | *          PID: 76
app_1     | * Listening on http://0.0.0.0:3000
app_1     | Use Ctrl-C to stop
```

6. Open the app in your browser

`http://[YOUR-SERVER-IP-ADDRESS]:3000`


7. Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>
Start again as daemon:

```console
docker-compose up -d
Starting solectrus_redis_1    ... done
Starting solectrus_db_1       ... done
Starting solectrus_influxdb_1 ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_app_1                ... done
```

This way the services are started in the background. To check if this works, reboot machine:

```console
sudo reboot
```

Wait a bit, then login again and check if Docker Compose has auto-started the services.

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus
docker ps

NAMES                            STATUS
solectrus_app_1                  Up 31 seconds (healthy)
solectrus_forecast-collector_1   Up 31 seconds
solectrus_influxdb_1             Up 33 seconds
solectrus_redis_1                Up 33 seconds
solectrus_db_1                   Up 33 seconds
```

8. Staying up to date

To update your installation to the latest release, run:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus

docker-compose pull
docker-compose up -d
```
