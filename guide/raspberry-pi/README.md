# Install SOLECTRUS on a Raspberry Pi

This guide demonstrates how to run **all** components of SOLECTRUS on a Raspberry Pi with ARM64 architecture:

- Dashboard (the SOLECTRUS UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database for storing settings like historical prices)
- SENEC Collector (pulling measurements from SENEC devise and pushing them the InfluxDB)
- Forecast-Collector (optional)

This guide is tested on a Raspberry Pi Model 4 with 8GB RAM using Raspberry Pi OS Lite (Debian Bullseye, 64bit). A Linux Kernel v4 or higher is **required**.

This machine connects to your SENEC device, so it should be placed in the same network.

0. Ensure you have Pi OS 64bit

Check your OS version:

```console
$ uname -a
Linux raspberrypi 5.15.84-v8+ #1613 SMP PREEMPT Thu Jan 5 12:03:08 GMT 2023 aarch64 GNU/Linux
```

The important part is `aarch64` which means you are running a 64bit OS. If you are running a 32bit OS, you need to upgrade.

The easiest way to upgrade is to use the Raspberry Pi Imager to install Raspberry Pi OS Lite (Debian Bullseye, 64bit) on a SD card:
https://www.raspberrypi.com/software/

1. Ensure Docker is installed and running.

Check your Docker version:

```console
$ docker --version
Docker version 23.0.0, build e92dd87

$ docker compose version
Docker Compose version v2.15.1
```

If you don't have Docker installed, follow the instructions here:
https://docs.docker.com/engine/install/debian/

2. Create folders for configuration and Docker volumes:

Choose a folder where you want to store the configuration and Docker volumes. This guide assumes you have a folder `/home/pi/solectrus` which is used for Docker volumes.

```console
ssh pi@[YOUR-RASPI-IP-ADDRESS]
cd /home/pi
mkdir -p solectrus
cd solectrus
mkdir redis postgresql influxdb
```

3. Download and prepare configuration file

Download the configuration file from the repository;

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/raspberry-pi/.env" -o /home/pi/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```console
nano .env
```

IMPORTANT settings, MUST be changed:

- SENEC_HOST # Hostname or IP address of your SENEC device
- APP_HOST # Hostname or IP address of your Raspberry Pi

Not required, but highly recommended:

- ADMIN_PASSWORD
- INSTALLATION_DATE
- ELECTRICITY_PRICE
- FEED_IN_TARIFF
- INFLUX_PASSWORD
- INFLUX_ADMIN_TOKEN
- FORECAST_LATITUDE
- FORECAST_LONGITUDE
- FORECAST_DECLINATION
- FORECAST_AZIMUTH
- FORECAST_KWP

Note: Editing the file with `nano` is just an example, because this editor is pre-installed. You can use any editor you like.

4. Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/raspberry-pi/docker-compose.yml" -o /home/pi/solectrus/docker-compose.yml
```

5. Start Docker containers

```console
docker compose up
```

This could take some minutes for the first run. You see some output like this:

```
.... lots of other lines here ....
app_1     | Created database 'solectrus_production'
app_1     | Database is ready!
app_1     | Puma starting in single mode...
app_1     | * Puma version: 6.0.2 (ruby 3.2.0-p0) ("Sunflower")
app_1     | *  Min threads: 5
app_1     | *  Max threads: 5
app_1     | *  Environment: production
app_1     | *          PID: 76
app_1     | * Listening on http://0.0.0.0:3000
app_1     | Use Ctrl-C to stop
```

6. Open the app in your browser

`http://[YOUR-RASPI-IP-ADDRESS]:3000`

7. Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>
Start again as daemon:

```console
docker compose up -d
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
ssh pi@[YOUR-RASPI-IP-ADDRESS]
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
ssh pi@[YOUR-RASPI-IP-ADDRESS]
cd solectrus

docker compose pull
docker compose up -d
```

Highly recommended: To automate installing updates, you can use [Watchtower](https://containrrr.dev/watchtower/), which is a free tool to automatically update running Docker containers. Once installed, it will check for new Docker images once every day and updates your containers automatically. And of course, Watchtower also runs in a Docker container.
