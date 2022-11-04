# Distributed installation on Raspberry Pi pushing to a remote server

This guides demonstrates how to set up a Solectrus instance on a remote server (STEP 1) and install the SENEC Collector on your local Raspberry Pi (STEP 2).

## STEP 1: Install Solectrus on a remote server

### a) Order your server at Hetzner

Signup on Hetzner:
https://hetzner.cloud/?ref=NggV8HU9FqCz
(referral link, will give you a discount of €20)

Order your server:

- Go to https://console.hetzner.cloud/projects
- Select "New project", name it "Solectrus", open the project
- Select "Add server"
- Location: Select a location near you
- Image: Select "Apps", then "Docker CE"
- Type: The smallest machine is enough, so select "CX11"
- SSH-Key: If you already have an SSH key, you can add it here to avoid struggling with SSH password. Otherwise, leave it blank.
- Order (for €4,15 per month)

### b) First Login

Wait for the email from Hetzner. Write down the IP address of your server. It will be referred later as [YOUR-SERVER-IP-ADDRESS].

Login to your server:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
```

If your are not using SSH keys, you are asked to enter your password. Type in our password you get from Hetzner per email. On first login, you are asked to change your password. Choose a strong password.

Check if `Docker` is installed and running:

```console
docker -v
Docker version 20.10.17, build 100c701
```

Nice, `Docker` is preinstalled, but we need to install `Docker Compose`, too:

```console
sudo curl -L "https://github.com/docker/compose/releases/download/v2.10.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker-compose -v
Docker Compose version v2.10.2
```

### c) Create folders for configuration and Docker volumes:

```console
mkdir solectrus
cd solectrus
mkdir redis postgresql influxdb
```

### d) Create configuration file

Download the configuration file from the repository;

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/external-server/.env" -o ~/solectrus/.env
```

Edit the downloaded file and change values to your needs.

```console
pico .env
```

You should change at least the following values:

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

Save file and close the editor: <kbd>Ctrl+S</kbd>, then <kbd>Ctrl+X</kbd>


### e) Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/step-by-step/external-server/docker-compose.yml" -o ~/solectrus/docker-compose.yml
```

### f) Start Docker containers

To check if all works fine, we start the containers in the foreground:

```console
docker-compose up
```

This could take some minutes for the first run, because some images are download. You see some output like this:

```
Creating network "solectrus_default" with the default driver
Pulling influxdb (influxdb:2.5.1)...
2.5.1: Pulling from library/influxdb
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

Note:

- Port `3000` (Dashboard UI) is mapped to 80, so you can access it directly from your browser
- Port `8086` (InfluxDB) is exposed to the outside, so it can be reached by the Raspberry (see Step 2)

### g) Open the app in your browser

Open `http://[YOUR-SERVER-IP-ADDRESS]` in your browser. You should see the dashboard.


### h) Check login to InfluxDB

Open `http://[YOUR-SERVER-IP-ADDRESS]:8086` in your browser. You should see the login of InfluxDB.

Login with `admin` and the password you defined in `INFLUX_PASSWORD` (see `.env` file)

### i) Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>. Then start again as daemon:

```console
docker-compose up -d
Starting solectrus_redis_1    ... done
Starting solectrus_db_1       ... done
Starting solectrus_influxdb_1 ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_app_1                ... done
```

To check if this works, reboot the machine:

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

### j) Finish!

You are done with step 1. Solectrus is now installed on your server and can be accessed from your browser.

`http://[YOUR-SERVER-IP-ADDRESS]`


### h) Staying up to date

To update your installation to the latest release, run:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus

docker-compose pull
docker-compose up -d
```

## STEP 2: Install SENEC Collector on a Raspberry Pi

### a) Prepare Raspberry Pi

Login to you Raspi and ensure that Docker is installed:

```console
ssh pi@raspberrypi.local
docker -v
Docker version 20.10.12, build e91ed57
```

If Docker is not installed, install it:

```console
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker Pi
```

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
         -e SENEC_INTERVAL=5 \
         -e INFLUX_PORT=8086 \
         -e INFLUX_HOST=[YOUR-SERVER-IP-ADDRESS] \
         -e INFLUX_ORG=solectrus \
         -e INFLUX_BUCKET=my-solectrus-bucket \
         -e INFLUX_TOKEN=my-super-secret-admin-token \
         ghcr.io/solectrus/senec-collector:latest
```

This will most likely be successful, but be sure to check the output:

```console
docker logs -f senec-collector
```

If it works, you should see this:

```
SENEC collector for SOLECTRUS
https://github.com/solectrus/senec-collector
Copyright (c) 2020,2022 Georg Ledermann, released under the MIT License

Using Ruby 3.1.2 on platform arm-linux-musleabihf
Pulling from SENEC at [YOUR-SENEC-IP-ADDRESS] every 5 seconds
Pushing to InfluxDB at http://[YOUR-SERVER-IP-ADDRESS]:8086, bucket my-solectrus-bucket


Got record #1 from SENEC: Inverter 4127 W, House 0 W, 2022-02-13 12:06:54 +0000
Successfully pushed record to InfluxDB
```

Great! Since the container runs in the background, it is automatically restarted at every reboot.

If something goes wrong, stop and remove the container first:

```console
docker stop senec-collector
docker rm senec-collector
```

Check the arguments of your `docker run` command:

- Is you SENEC devise responding to the IP address you defined as `SENEC_HOST`?
- Is the InfluxDB server (see Step 1) responding to the IP address you defined as `INFLUX_HOST`?
- Did your Influx credentials given in `INFLUX_ORG`, `INFLUX_BUCKET` and `INFLUX_TOKEN` match your InfluxDB setup (see Step 1)?

Change them until pulling and pushing works.


### c) Finish!

You are done. You should see the measurements from your SENEC device in your Solectrus instance:

`http://[YOUR-SERVER-IP-ADDRESS]`
