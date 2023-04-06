# Install SOLECTRUS on an in-house server

This guide demonstrates how to run **all** components of SOLECTRUS on a single machine:

- Dashboard (the SOLECTRUS UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database for storing settings like historical prices)
- SENEC Collector (pulling measurements from SENEC devise and pushing them the InfluxDB)
- Forecast-Collector (optional)

This guide is tested on a Synology NAS DS220+, but should work on any Linux system with `x86_64` architecture which can run Docker. A CPU with at least 2 cores is recommended, as well as a RAM upgrade to more than 2GB. A Linux Kernel v4 or higher is **required**, some older NAS devices don't work because they are on Kernel v3 and cannot be updated.

This machine connects to your SENEC device, so it should be placed in the same network.

1. Ensure Docker and Docker Compose is installed and running. Docker 18.06.0 or later is required.

```console
docker --version
Docker version 20.10.3, build 55f0773

docker-compose -v
docker-compose version 1.28.5, build 24fb474e
```

To ensure Docker can run without `sudo`, create a `docker` group and add your user to it:

```console
sudo synogroup --add docker
sudo chown root:docker /var/run/docker.sock
sudo synogroup --member docker $USER
```

(adopted from https://docs.docker.com/engine/install/linux-postinstall/)

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
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/synology/.env" -o /volume1/docker/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```console
vim .env
```

IMPORTANT settings, MUST be changed:

- SENEC_HOST # Hostname or IP address of your SENEC device
- APP_HOST # Hostname or IP address of your Synology NAS

Not required, but highly recommended:

- ADMIN_PASSWORD
- INSTALLATION_DATE
- ELECTRICITY_PRICE
- INFLUX_PASSWORD
- INFLUX_ADMIN_TOKEN
- FEED_IN_TARIFF
- FORECAST_LATITUDE
- FORECAST_LONGITUDE
- FORECAST_DECLINATION
- FORECAST_AZIMUTH
- FORECAST_KWP

Note: Editing the file with `vim` is just an example, because this is the pre-installed editor. You can use any editor you like. If you are not familiar with `vim`, you can use [TextEditor](https://www.synology.com/dsm/packages/TextEditor) by Synology which can be installed as package via DSM.

4. Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/synology/docker-compose.yml" -o /volume1/docker/solectrus/docker-compose.yml
```

5. Start Docker containers

```console
docker-compose up
```

This could take some minutes for the first run. You see some output like this:

```
Creating network "dashboard_default" with the default driver
Creating dashboard_db_1       ... done
Creating dashboard_influxdb_1 ... done
Creating dashboard_redis_1    ... done
..... [Pausing here to wait for the databases to be created] .....
Creating dashboard_forecast-collector_1 ... done
Creating dashboard_senec-collector_1    ... done
Creating dashboard_app_1                ... done
Attaching to dashboard_db_1, dashboard_redis_1, dashboard_influxdb_1, dashboard_forecast-collector_1, dashboard_senec-collector_1, dashboard_app_1
app_1                 | Starting SOLECTRUS...
app_1                 | Version: v0.10.0 - 2023-04-01T17:08:24+02:00
app_1                 | ----------------
app_1                 | influxdb (172.24.0.4:8086) open
app_1                 | InfluxDB is up and running!
app_1                 | db (172.24.0.2:5432) open
app_1                 | PostgreSQL is up and running!
app_1                 | Preparing database...
.....
app_1                 | Created database 'solectrus_production'
app_1                 | Database is ready!
app_1                 | Puma starting in single mode...
app_1                 | * Puma version: 6.2.1 (ruby 3.2.2-p53) ("Speaking of Now")
app_1                 | *  Min threads: 5
app_1                 | *  Max threads: 5
app_1                 | *  Environment: production
app_1                 | *          PID: 11
app-1                 | * Listening on http://0.0.0.0:3001
app-1                 | Use Ctrl-C to stop
.....
senec-collector_1     |
senec-collector_1     | Got record #1 from SENEC at 192.168.178.123: PV + ENTLADEN, Inverter 1169 W, House 2518 W, 2023-04-06 13:36:57 +0000
senec-collector_1     | Successfully pushed record #1 to InfluxDB
senec-collector_1     |
senec-collector_1     | Got record #2 from SENEC at 192.168.178.123: PV + ENTLADEN, Inverter 1164 W, House 2509 W, 2023-04-06 13:37:02 +0000
senec-collector_1     | Successfully pushed record #2 to InfluxDB
senec-collector_1     |
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

8. Optional: Import historical data

On [mein-senec.de](https://mein-senec.de) you find download links for your historical data. Download the CSV files and import them into SOLECTRUS. See [SENEC-Importer](https://github.com/solectrus/senec-importer) for more information.

9. Staying up to date

To update your installation to the latest release, run:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus

docker-compose pull
docker-compose up -d
```

Highly recommended: To automate installing updates, you can use [Watchtower](https://containrrr.dev/watchtower/), which is a free tool to automatically update running Docker containers. Once installed, it will check for new Docker images once every day and updates your containers automatically. And of course, Watchtower also runs in a Docker container.
