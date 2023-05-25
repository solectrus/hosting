# Install and setup SOLECTRUS with evcc

This guide demonstrates how to run **all** components of SOLECTRUS on a Linux box:

- Dashboard (the SOLECTRUS UI)
- InfluxDB (the database for storing the measurements)
- Redis (for some caching)
- PostgreSQL (the SQL database for storing settings like historical prices)
- MQTT Collector (getting measurements via MQTT and pushing them to the InfluxDB)
- Forecast-Collector (optional)

## Step 0: Prepare evcc

- Ensure that `evcc` is up and running
- Set up an MQTT broker that can be local or in the cloud (such as https://www.hivemq.com/public-mqtt-broker/)
- Configure `evcc` to send measurements to the broker: https://docs.evcc.io/docs/reference/configuration/mqtt/

## Step 1: Check your Linux box

A 64bit Linux with Kernel v4 or higher and AMD64 or ARM64 architecture is **required**. You can check your OS and architecture with the following commands:

```console
$ uname -a
Linux raspberrypi 6.1.21-v8+ #1642 SMP PREEMPT Mon Apr  3 17:24:16 BST 2023 aarch64 GNU/Linux

$ dpkg --print-architecture
arm64
```

This example output shows:

- The kernel is v6, which is the latest and greatest - v4 or v5 will work as well.

- The architecture is `arm64` which means you are running a 64bit OS. If you are running a 32bit OS, you need to upgrade. If the architecture is `armhf`, you are running a 64bit Kernel with 32bit userland, which will **not** work.

## Step 2. Ensure Docker is installed and running.

Check your Docker version:

```console
$ docker --version
Docker version 24.0.1, build 6802122

$ docker compose version
Docker Compose version v2.18.1
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

## Step 3: Create folders for configuration and Docker volumes:

Choose a folder where you want to store the configuration and Docker volumes. This guide assumes you have a folder `/home/pi/solectrus` which is used for Docker volumes.

```console
ssh pi@[YOUR-LINUXBOX-IP]
cd /home/pi
mkdir -p solectrus
cd solectrus
mkdir redis postgresql influxdb
```

3. Download and prepare configuration file

Download the configuration file from the repository;

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/mqtt-evcc/.env" -o /home/pi/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```console
nano .env
```

Please check all settings, especially the following:

- APP_HOST # Hostname or IP address of your Linux box
- all MQTT\_\* settings

Note: Editing the file with `nano` is just an example, because this editor is pre-installed. You can use any editor you like.

4. Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/mqtt-evcc/docker-compose.yml" -o /home/pi/solectrus/docker-compose.yml
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
✔ Container solectrus-mqtt-collector-1      Created
Attaching to solectrus-app-1, solectrus-db-1, solectrus-forecast-collector-1, solectrus-influxdb-1, solectrus-redis-1, solectrus-mqtt-collector-1
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
solectrus-mqtt-collector-1      | MQTT collector for SOLECTRUS, Version 0.1.0, built at 2023-05-22T11:02:51.907Z
solectrus-mqtt-collector-1      | https://github.com/solectrus/mqtt-collector
solectrus-mqtt-collector-1      | Copyright (c) 2023 Georg Ledermann and contributors, released under the MIT License
solectrus-mqtt-collector-1      |
solectrus-mqtt-collector-1      | Using Ruby 3.2.2 on platform aarch64-linux-musl
solectrus-mqtt-collector-1      | Subscribing from MQTT broker at mqtt://my-iobroker-host:1883
solectrus-mqtt-collector-1      | Pushing to InfluxDB at http://localhost:8086, bucket my-bucket
solectrus-mqtt-collector-1      |
solectrus-mqtt-collector-1      | 2023-05-25 14:31:33 +0000 {"bat_power_plus"=>0, "bat_power_minus"=>17}
solectrus-mqtt-collector-1      | 2023-05-25 14:31:33 +0000 {"bat_fuel_charge"=>100.0}
solectrus-mqtt-collector-1      | 2023-05-25 14:31:33 +0000 {"grid_power_plus"=>0, "grid_power_minus"=>788}
solectrus-mqtt-collector-1      | 2023-05-25 14:31:33 +0000 {"house_power"=>330}
solectrus-mqtt-collector-1      | 2023-05-25 14:31:33 +0000 {"inverter_power"=>1101}
....
```

6. Open the app in your browser

`http://[YOUR-LINUXBOX-IP]:3000`

7. Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>
Start again as daemon:

```console
docker compose up -d
Starting solectrus_redis_1    ... done
Starting solectrus_db_1       ... done
Starting solectrus_influxdb_1 ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_mqtt-collector_1     ... done
Starting solectrus_app_1                ... done
```

This way the services are started in the background. To check if this works, reboot machine:

```console
sudo reboot
```

Wait a bit, then login again and check if Docker Compose has auto-started the services.

```console
ssh pi@[YOUR-LINUXBOX-IP]
cd solectrus
docker ps

IMAGE                                         STAT1US                  NAMES
ghcr.io/solectrus/solectrus:latest            Up 31 second (healthy)   solectrus-app-1
ghcr.io/solectrus/forecast-collector:latest   Up 31 second             solectrus-forecast-collector-1
ghcr.io/solectrus/mqtt-collector:latest       Up 31 second             solectrus-mqtt-collector-1
influxdb:2.7-alpine                           Up 31 second (healthy)   solectrus-influxdb-1
redis:7-alpine                                Up 31 second (healthy)   solectrus-redis-1
postgres:15-alpine                            Up 31 second (healthy)   solectrus-db-1
```

8. Staying up to date

To update your installation to the latest release, run:

```console
ssh pi@[YOUR-LINUXBOX-IP]
cd solectrus

docker compose pull
docker compose up -d
```

Highly recommended: To automate installing updates, you can use [Watchtower](https://containrrr.dev/watchtower/), which is a free tool to automatically update running Docker containers. Once installed, it will check for new Docker images once every day and updates your containers automatically. And of course, Watchtower also runs in a Docker container.
