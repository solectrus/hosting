# Distributed installation on Raspberry Pi pushing to a remote server

This guides demonstrates how to set up a SOLECTRUS instance on a remote server (STEP 1) and install the SENEC Collector on your local Raspberry Pi (STEP 2).

## STEP 1: Install SOLECTRUS on a remote server

### a) Order your server at Hetzner

Sign up on Hetzner:
https://hetzner.cloud/?ref=NggV8HU9FqCz
(referral link, will give you a discount of €20 - and me too)

Order your server:

- Go to https://console.hetzner.cloud/projects
- Select "New project", name it "SOLECTRUS", open the project
- Select "Add server"
- Location: Select a location near you
- Image: Select "Apps", then "Docker CE"
- Type: The smallest machine is enough, so select "CX11" (x86) or "CAX11" (Arm64)
- SSH-Key: If you already have an SSH key, you can add it here to avoid struggling with SSH password. Otherwise, leave it blank.
- Order (for less than 5€ per month)

### b) First Login

After the server has been created, it receives a public IP address. Write it down, it will be referred later [YOUR-SERVER-IP-ADDRESS].

Login to your server:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
```

If your are not using SSH keys, you are asked to enter your password. Type in our password you get from Hetzner per email. On first login, you are asked to change your password. Choose a strong password.

Check if `Docker` is installed and running:

```console
docker --version
Docker version 25.0.1, build 29cf629

docker compose version
Docker Compose version v2.24.2
```

Nice, `Docker` is preinstalled.

### c) Create folders for configuration and Docker volumes:

```console
mkdir solectrus
cd solectrus
mkdir redis postgresql influxdb
```

### d) Create configuration file

Download the configuration file from the repository;

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/external-server/.env" -o .env
```

Edit the downloaded file and change values to your needs.

```console
pico .env
```

IMPORTANT settings, MUST be changed:

- APP_HOST # Hostname or IP address of your cloud server

Not required, but highly recommended:

- ADMIN_PASSWORD
- INSTALLATION_DATE
- INFLUX_PASSWORD
- INFLUX_ADMIN_TOKEN
- FORECAST_LATITUDE
- FORECAST_LONGITUDE
- FORECAST_DECLINATION
- FORECAST_AZIMUTH
- FORECAST_KWP

WARNING: Don't forget to change passwords, because otherwise everyone can access your database!

Save file and close the editor: <kbd>Ctrl+S</kbd>, then <kbd>Ctrl+X</kbd>

### e) Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/external-server/docker-compose.yml" -o docker-compose.yml
```

### f) Start Docker containers

To check if all works fine, we start the containers in the foreground:

```console
docker compose up
```

This could take some minutes for the first run, because some images are download. You see some output like this:

```
✔ Network solectrus_default                 Created
✔ Container solectrus-db-1                  Created
✔ Container solectrus-influxdb-1            Created
✔ Container solectrus-redis-1               Created
✔ Container solectrus-app-1                 Created
✔ Container solectrus-forecast-collector-1  Created
✔ Container solectrus-senec-collector-1     Created
Attaching to solectrus-app-1, solectrus-db-1, solectrus-forecast-collector-1, solectrus-influxdb-1, solectrus-redis-1
....
app-1                 | Starting SOLECTRUS...
app-1                 | Version: v0.14.2 - 2024-01-07T19:02:01+01:00 - v0.14.2
app-1                 | ----------------
app-1                 | InfluxDB is up and running!
app-1                 | influxdb (172.18.0.2:8086) open
app-1                 | PostgreSQL is up and running!
app-1                 | Preparing database...
app-1                 | db (172.18.0.3:5432) open
....
app-1                 | Created database 'solectrus_production'
app-1                 | Database is ready!
app-1                 | => Booting Puma
app-1                 | => Rails 7.1.2 application starting in production
app-1                 | => Run `bin/rails server --help` for more startup options
app-1                 | Puma starting in single mode...
app-1                 | * Puma version: 6.4.1 (ruby 3.2.2-p53) ("The Eagle of Durango")
app-1                 | *  Min threads: 5
app-1                 | *  Max threads: 5
app-1                 | *  Environment: production
app-1                 | *          PID: 1
app-1                 | * Listening on http://0.0.0.0:3000
app-1                 | Use Ctrl-C to stop
```

Note:

- Port `3000` (Dashboard UI) is mapped to 80, so you can access it from your browser without specifying the port
- Port `8086` (InfluxDB) is exposed to the outside, so it can be reached by the Raspberry (see Step 2)

### g) Open the app in your browser

Open `http://[YOUR-SERVER-IP-ADDRESS]` in your browser. You should see the dashboard.

### h) Check login to InfluxDB

Open `http://[YOUR-SERVER-IP-ADDRESS]:8086` in your browser. You should see the login of InfluxDB.

Login with `admin` and the password you defined in `INFLUX_PASSWORD` (see `.env` file)

### i) Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>. Then start again as daemon:

```console
docker compose up -d
Starting solectrus_redis_1              ... done
Starting solectrus_db_1                 ... done
Starting solectrus_influxdb_1           ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_app_1                ... done
```

To check if this works, reboot the machine:

```console
reboot
```

Wait a bit, then login again and check if Docker Compose has auto-started the services.

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus
docker ps


IMAGE                                         STATUS
ghcr.io/solectrus/solectrus:latest            Up 31 seconds (healthy) ...
ghcr.io/solectrus/forecast-collector:latest   Up 31 seconds           ...
influxdb:2.7-alpine                           Up 31 seconds (healthy) ...
postgres:16-alpine                            Up 31 seconds (healthy) ...
redis:7-alpine                                Up 31 seconds (healthy) ...
```

### j) Finish!

You are done with step 1. SOLECTRUS is now installed on your server and can be accessed from your browser.

`http://[YOUR-SERVER-IP-ADDRESS]`

### h) Optional: Import historical data

On [mein-senec.de](https://mein-senec.de) you find download links for your historical data. Download the CSV files and import them into SOLECTRUS. See [CSV-Importer](https://github.com/solectrus/csv-importer) for more information.

### i) Staying up to date

To update your installation to the latest release, run:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus

docker compose pull
docker compose up -d
```

To not have to do this manually every time, the `docker-compose.yml` contains [Watchtower](https://containrrr.dev/watchtower/), which is a free tool to automatically update running Docker containers. Once installed, it will check for new Docker images once every day and updates your containers automatically. And of course, Watchtower also runs in a Docker container.

## STEP 2: Install SENEC Collector on a Raspberry Pi

### a) Prepare Raspberry Pi

Login to you Raspi and ensure that Docker is installed:

```console
ssh pi@raspberrypi.local
docker --version
Docker version 25.0.1, build 29cf629
```

If Docker is not installed, install it:

```console
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Don't forget to add your user to the `docker` group, so you don't need to use `sudo` for every Docker command.

```console
sudo groupadd docker
sudo usermod -aG docker $USER
```

Log out and log back in so that your group membership is re-evaluated.

Details:
https://docs.docker.com/engine/install/linux-postinstall/

We don't need Docker Compose on the Raspi, because just one container will be started.

### b) Install and run SENEC Collector

To run the SENEC collector, first try to start the Docker container in the foreground:

IMPORTANT: Change the environment variables to match your SENEC device and your InfluxDB server from step 1!

```console
docker run \
         --name senec-collector \
         --privileged \
         --restart unless-stopped \
         -d \
         -e SENEC_HOST=[YOUR-SENEC-IP-ADDRESS] \
         -e SENEC_SCHEMA=[http OR https] \
         -e SENEC_INTERVAL=5 \
         -e SENEC_LANGUAGE=de \
         -e INFLUX_PORT=8086 \
         -e INFLUX_HOST=[YOUR-SERVER-IP-ADDRESS] \
         -e INFLUX_ORG=solectrus \
         -e INFLUX_BUCKET=solectrus \
         -e INFLUX_TOKEN=my-super-secret-admin-token \
         ghcr.io/solectrus/senec-collector:latest
```

This will most likely be successful, but be sure to check the output:

```console
docker logs -f senec-collector
```

If it works, you should see this:

```
SENEC collector for SOLECTRUS, Version 0.7.1, built at 2023-05-04T10:07:21.945Z
https://github.com/solectrus/senec-collector
Copyright (c) 2020,2023 Georg Ledermann, released under the MIT License
Using Ruby 3.2.2 on platform aarch64-linux-musl

Pulling from SENEC at [YOUR-SENEC-IP-ADDRESS] every 5 seconds
Pushing to InfluxDB at http://[YOUR-SERVER-IP-ADDRESS]:8086, bucket solectrus

Getting state names from [YOUR-SENEC-IP-ADDRESS] by parsing source code...
OK, got 99 state names

Got record #1 from SENEC at [YOUR-SENEC-IP-ADDRESS]: LADEN, Inverter 307 W, House 337 W, 2023-04-07 06:59:47 +0000
Successfully pushed record #1 to InfluxDB
```

Great! Since the container runs in the background, it is automatically restarted at every reboot.

If something goes wrong, stop and remove the container first:

```console
docker stop senec-collector
docker rm senec-collector
```

Check the arguments of your `docker run` command:

- Is you SENEC device responding to the IP address you defined as `SENEC_HOST`?
- Have you defined the correct `SENEC_SCHEMA` (http or https)? It's the same as you use in your browser to access your SENEC device.
- Is the InfluxDB server (see Step 1) responding to the IP address you defined as `INFLUX_HOST`?
- Did your Influx credentials given in `INFLUX_ORG`, `INFLUX_BUCKET` and `INFLUX_TOKEN` match your InfluxDB setup (see Step 1)?

Change them until pulling and pushing works.

### c) Finish!

You are done. You should see the measurements from your SENEC device in your SOLECTRUS instance:

`http://[YOUR-SERVER-IP-ADDRESS]`
