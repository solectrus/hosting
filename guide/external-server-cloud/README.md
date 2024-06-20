# Remote installation with cloud access to SENEC

This guides demonstrates how to set up a SOLECTRUS instance on a remote server at Hetzner with cloud access to SENEC.
**Please note: Cloud access is new and not yet tested by many users.**

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
- Type: The smallest machine is enough, so select "Arm64" architecture and then "CAX11"
- SSH-Key: If you already have an SSH key, you can add it here to avoid struggling with SSH password. Otherwise, leave it blank.
- Order (for €4,51 per month)

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
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/external-server-cloud/.env" -o .env
```

Edit the downloaded file and change values to your needs.

```console
pico .env
```

IMPORTANT settings, MUST be changed:

- APP_HOST # Hostname or IP address of your cloud server
- SENEC_USERNAME # Your SENEC username
- SENEC_PASSWORD # Your SENEC password
- SENEC_SYSTEM_ID # Your SENEC system ID (optional, if you have multiple systems)

Not required, but highly recommended:

- ADMIN_PASSWORD
- INSTALLATION_DATE
- INFLUX_PASSWORD
- INFLUX_ADMIN_TOKEN
- FORECAST_PROVIDER (and related forecast variables)

Save file and close the editor: <kbd>Ctrl+S</kbd>, then <kbd>Ctrl+X</kbd>

### e) Download Docker compose file `./docker-compose.yml`

```console
curl -L "https://raw.githubusercontent.com/solectrus/hosting/main/guide/external-server-cloud/docker-compose.yml" -o docker-compose.yml
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
Attaching to solectrus-app-1, solectrus-db-1, solectrus-forecast-collector-1, solectrus-senec-collector-1, solectrus-influxdb-1, solectrus-redis-1
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

### g) Open the app in your browser

Open `http://[YOUR-SERVER-IP-ADDRESS]` in your browser. You should see the dashboard.

### h) Run services in the background

Stop services by pressing <kbd>Ctrl+C</kbd>. Then start again as daemon:

```console
docker compose up -d
Starting solectrus_redis_1              ... done
Starting solectrus_db_1                 ... done
Starting solectrus_influxdb_1           ... done
Starting solectrus_forecast-collector_1 ... done
Starting solectrus_senec-collector_1    ... done
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
ghcr.io/solectrus/senec-collector:latest      Up 31 seconds           ...
influxdb:2.7-alpine                           Up 31 seconds (healthy) ...
postgres:16-alpine                            Up 31 seconds (healthy) ...
redis:7-alpine                                Up 31 seconds (healthy) ...
```

### i) Finish!

You are done with step 1. SOLECTRUS is now installed on your server and can be accessed from your browser.

`http://[YOUR-SERVER-IP-ADDRESS]`

### j) Optional: Import historical data

On [mein-senec.de](https://mein-senec.de) you find download links for your historical data. Download the CSV files and import them into SOLECTRUS. See [CSV-Importer](https://github.com/solectrus/csv-importer) for more information.

### k) Staying up to date

To update your installation to the latest release, run:

```console
ssh root@[YOUR-SERVER-IP-ADDRESS]
cd solectrus

docker compose pull
docker compose up -d
```

To not have to do this manually every time, the `docker-compose.yml` contains [Watchtower](https://containrrr.dev/watchtower/), which is a free tool to automatically update running Docker containers. Once installed, it will check for new Docker images once every day and updates your containers automatically. And of course, Watchtower also runs in a Docker container.

### l) Finish!

You are done. You should see the measurements from your SENEC device in your SOLECTRUS instance:

`http://[YOUR-SERVER-IP-ADDRESS]`

### m) Optional: secure your SOLECTRUS installation

If you want to use `https` to access your SOLECTRUS installation, please check out the separate [installation instructions](../external-server/README-https.md), simply ignore the section that talks about configuring `senec-collector`.
