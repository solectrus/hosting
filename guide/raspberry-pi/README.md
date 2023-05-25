# Install SOLECTRUS on a Raspberry Pi

This guide demonstrates how to run **all** components of SOLECTRUS on a Raspberry Pi with ARM64 architecture:

- Dashboard (the SOLECTRUS UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database for storing settings like historical prices)
- SENEC Collector (pulling measurements from SENEC device and pushing them to the InfluxDB)
- Forecast-Collector (optional)

This guide is tested on a Raspberry Pi Model 4 with 8GB RAM using Raspberry Pi OS Lite (Debian Bullseye, 64bit). A Linux Kernel v4 or higher is **required**.

This machine connects to your SENEC device, so it should be placed in the same network.

0. Ensure you have Pi OS 64bit with Kernel v4 or higher

Check your OS and architecture, which should look like this:

```console
$ uname -a
Linux raspberrypi 6.1.21-v8+ #1642 SMP PREEMPT Mon Apr  3 17:24:16 BST 2023 aarch64 GNU/Linux

$ dpkg --print-architecture
arm64
```

The kernel is v6, which is the latest and greatest - v4 or v5 will work as well.

The architecture is `arm64` which means you are running a 64bit OS. If you are running a 32bit OS, you need to upgrade. If the architecture is `armhf`, you are running a 64bit Kernel with 32bit userland, which will **not** work.

The easiest way to setup a brand new OS is to use the Raspberry Pi Imager to install Raspberry Pi OS Lite (Debian Bullseye, 64bit) on a SD card:
https://www.raspberrypi.com/software/

1. Ensure Docker is installed and running.

Check your Docker version:

```console
$ docker --version
Docker version 23.0.6, build ef23cbc

$ docker compose version
Docker Compose version v2.17.3
```

If you don't have Docker installed, follow the instructions here:
https://docs.docker.com/engine/install/debian/

Don't forget to add your user to the `docker` group, so you don't need to use `sudo` for every Docker command.

```console
sudo groupadd docker
sudo usermod -aG docker $USER
```

Log out and log back in so that your group membership is re-evaluated.

Details:
https://docs.docker.com/engine/install/linux-postinstall/

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
✔ Network solectrus_default                 Created
✔ Container solectrus-db-1                  Created
✔ Container solectrus-influxdb-1            Created
✔ Container solectrus-redis-1               Created
✔ Container solectrus-app-1                 Created
✔ Container solectrus-forecast-collector-1  Created
✔ Container solectrus-senec-collector-1     Created
Attaching to solectrus-app-1, solectrus-db-1, solectrus-forecast-collector-1, solectrus-influxdb-1, solectrus-redis-1, solectrus-senec-collector-1
....
solectrus-app-1                 | Starting SOLECTRUS...
solectrus-app-1                 | Version: v0.10.1 - 2023-05-06T09:46:43+02:00
solectrus-app-1                 | ----------------
solectrus-app-1                 | influxdb (172.18.0.3:8086) open
solectrus-app-1                 | InfluxDB is up and running!
solectrus-app-1                 | db (172.18.0.4:5432) open
solectrus-app-1                 | PostgreSQL is up and running!
solectrus-app-1                 | Preparing database...
....
solectrus-app-1                 | Created database 'solectrus_production'
solectrus-app-1                 | Database is ready!
solectrus-app-1                 | Puma starting in single mode...
solectrus-app-1                 | * Puma version: 6.2.2 (ruby 3.2.2-p53) ("Speaking of Now")
solectrus-app-1                 | *  Min threads: 5
solectrus-app-1                 | *  Max threads: 5
solectrus-app-1                 | *  Environment: production
solectrus-app-1                 | *          PID: 11
solectrus-app-1                 | * Listening on http://0.0.0.0:3001
solectrus-app-1                 | Use Ctrl-C to stop
....
solectrus-senec-collector-1     |
solectrus-senec-collector-1     | Got record #1 from SENEC at 192.168.178.29: AKKU VOLL, Inverter 757 W, House 492 W, 2023-04-06 14:25:16 +0000
solectrus-senec-collector-1     | Successfully pushed record #1 to InfluxDB
solectrus-senec-collector-1     |
solectrus-senec-collector-1     | Got record #2 from SENEC at 192.168.178.29: AKKU VOLL, Inverter 753 W, House 503 W, 2023-04-06 14:25:21 +0000
solectrus-senec-collector-1     | Successfully pushed record #2 to InfluxDB
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

IMAGE                                         STAT1US                  NAMES
ghcr.io/solectrus/solectrus:latest            Up 31 second (healthy)   solectrus-app-1
ghcr.io/solectrus/forecast-collector:latest   Up 31 second             solectrus-forecast-collector-1
ghcr.io/solectrus/senec-collector:latest      Up 31 second             solectrus-senec-collector-1
influxdb:2.7-alpine                           Up 31 second (healthy)   solectrus-influxdb-1
redis:7-alpine                                Up 31 second (healthy)   solectrus-redis-1
postgres:15-alpine                            Up 31 second (healthy)   solectrus-db-1
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
