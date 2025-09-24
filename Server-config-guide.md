# VM config
6144MB RAM
32GB SSD
4 Cores

# Ubuntu Server Installation
```JSON
Setup Configuration {
    "Language": "English",
    "Update Installer": True,
    "Keyboard" : "German - German",
    "Install-Type": "Normal",
    "Search 3rd party driver": True,
    "Disk as LVM group" : False,
    "User" : {
        "Full Name" : "LF10b",
        "Servername" : "main",
        "username" : "lf10b",
        "password" : "lf10b"
    },
    "Ubuntu Pro" : False,
    "OpenSSH Server" : True
}
```

# Ports used
Port | Service
--- | ---
25565 | Minecraft Server (Pterodactyl)
9090 | Prometheus
3000 | Grafana
9100 | node exporter

## System Update
nach reboot ausführen:
`sudo apt update && sudo apt upgrade -y`

# Pterodactyl Installation
<!-- Guide: https://pterodactyl.io/panel/1.0/getting_started.html -->
<!-- might be smart to run as root-user -->
## Dependency installation
```BASH

sudo apt install net-tools -y
sudo apt install tree -y

# Add "add-apt-repository" command
sudo apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg

# Add additional repositories for PHP (Ubuntu 22.04)
sudo LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php

# Add Redis official APT repository
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# Update repositories list
apt update
sudo apt upgrade

# Install Dependencies
sudo apt -y install php8.3 php8.3-{common,cli,gd,mysql,mbstring,bcmath,xml,fpm,curl,zip} mariadb-server nginx tar unzip git redis-server
```

## Install Composer
Composer = dependency manager for PHP
```bash
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
```

## Download needed Files
move to Folder
```bash
sudo mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl
```
Download pre-packaged content and unpack it and set correct permission
```bash
sudo curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
sudo tar -xzvf panel.tar.gz
sudo chmod -R 755 storage/* bootstrap/cache/
```

## Installation
### DB Configuration
logon to mysql as root
Password is empty
```bash
sudo mysql -u root
```
```SQL
--Creating a user
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'pterodactyl';
--Create a Database
CREATE DATABASE panel;
-- Assigning permissions
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1';
exit
```

### Copy default environment
```bash
cd /var/www/pterodactyl
sudo cp .env.example .env
sudo COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
# Only run the command below if you are installing this Panel for
# the first time and do not have any Pterodactyl Panel data in the database.
sudo php artisan key:generate --force
```

Backup encryption key 
```bash
grep APP_KEY /var/www/pterodactyl/.env > /home/lf10b/APIKEY_backup
```

Environment Config
```bash
sudo php artisan p:environment:setup
sudo php artisan p:environment:database
```

```JSON
Config {
    "Author Mail" : "example@exchange.com",
    "Application URL" : "http://192.168.65.161/panel",
    "Timezone": "UTC",
    "Cache Driver" : "redis",
    "Session Driver" : "redis",
    "queue Driver" : "redis",
    "UI Settings Editor" : True,
    "Telemtry Data" : False,
    "Redis Host" : "127.0.0.1",
    "Redis Password" : None,
    "Redis Port" : 6379,
    "Database Host" : "127.0.0.1",
    "Database Port" : 3306,
    "Database Name" : "panel",
    "Database Username" : "pterodactyl",
    "Database Password" : "pterodactyl"
}
```

May takae some time
```bash
# Database Setup
sudo php artisan migrate --seed --force
# Add the first user
sudo php artisan p:user:make
```
```JSON
FirstUser {
    "IsAdmin": True,
    "Mail": "boblukulus@outlook.com",
    "username" : "user1",
    "First Name": "User",
    "Last Name": "one",
    "Password": "Password_01"
}
```
```bash
# Set Permissions
sudo chown -R www-data:www-data /var/www/pterodactyl/*
```

## Queue Listeners
The first thing we need to do is create a new cronjob that runs every minute to process specific Pterodactyl tasks, such as session cleanup and sending scheduled tasks to daemons. You'll want to open your crontab using sudo crontab -e and then paste the line below.
```sudo crontab -e````
Insert into file:
```crontab
* * * * * php /var/www/pterodactyl/artisan schedule:run >> /dev/null 2>&1
```

### Create Queue Worker
Create a service file
`sudo touch /etc/systemd/system/pteroq.service`
Fill with
```servicefile
# Pterodactyl Queue Worker File
# ----------------------------------

[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
# On some systems the user and group might be different.
# Some systems use `apache` or `nginx` as the user and group.
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable Redis on Startup up
`sudo systemctl enable --now redis-server`

Enable pteroq service on startup
`sudo systemctl enable --now pteroq.service`

## Webserver Config
remove default nginx config
`rm /etc/nginx/sites-enabled/default`

Create server config
```bash
sudo nano /etc/nginx/sites-available/pterodactyl.conf
````

```config
server {
    # Replace the example <domain> with your domain name or IP address
    listen 80;
    server_name panel;

    root /var/www/pterodactyl/public;
    index index.html index.htm index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### Enabling Configuration
Restart Nginx and symlink the file
```bash
sudo ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/pterodactyl.conf
sudo systemctl restart nginx
```

## Install Wings
### Dependencies
#### Docker installation
Docker CE will be used
```bash
# Quick install docker
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
#Start Docker on Startup
sudo systemctl enable --now docker
```
### Actual Install
First get the required directory structure setup
```bash
sudo mkdir -p /etc/pterodactyl
sudo curl -L -o /usr/local/bin/wings "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_$([[ "$(uname -m)" == "x86_64" ]] && echo "amd64" || echo "arm64")"
sudo chmod u+x /usr/local/bin/wings
```

### Configure
On admin Panel go to location > create new location
```JSON
locationDetail {
    "Short Code" : "main-location"
}
```

then create new node
go to nodes > Create New
```JSON
NodeSettings {
    "Node Name" : "minecraft",
    "Description" : "Minecraft Server Node",
    "Location" : "main-location",
    "Node Visibility" : "Public",
    "FQDN" : "192.168.65.161",
    "SSL" : False,
    "Behind Proxy": False,
    "Total Memory" : 4096,
    "Overallocate" : -1,
    "Disk Space" : 8192,
    "Overallocate" : -1,
    "Daemon Port" : 8080,
    "Daemon SFTP Port" : 2022
}
```

Open Node go to Configuration and copy Configuration File's content

```bash
sudo nano /etc/pterodactyl/config.yml
```

paste content
```yml
debug: false
uuid: 3a4b8dc5-81e9-446d-a4d1-1cd01a6adb89
token_id: gI0maK0eMwbTGXbn
token: lTKtAlRo9whpv5PJk0hXzCoTnxdJUz2kfxgafRTgn4Aq9pMAdMW5pvm8J5xOobgc
api:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: false
    cert: /etc/letsencrypt/live/192.168.65.161/fullchain.pem
    key: /etc/letsencrypt/live/192.168.65.161/privkey.pem
  upload_limit: 100
system:
  data: /var/lib/pterodactyl/volumes
  sftp:
    bind_port: 2022
allowed_mounts: []
remote: 'http://192.168.65.161'
```

### Start Wings
Start Wings and confirm its running without errors then exit with ^C

```sudo wings --debug```

#### Daemonize it
```bash
sudo nano /etc/systemd/system/wings.service
```

paste:
```service
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable Service
```bash
sudo systemctl enable --now wings
```

### Node Allocations
Nodes > minecraft > Allocation (Tab)
To get all IPs from all Interfaces who are connected to Internet use `ip addr | grep "inet "`
```JSON
AssignNewAllocation {
    "IP Address" : "192.168.65.161",
    "Ports" : "25565-25580" //Ports used by Minecraft
}
```

# Minecraft Installation
## Open Firewall Ports
```bash
# Run as root user
sudo su -
# Enbale the ports
ufw allow 25565:25580/tcp
```

## Create Server
On Dashboard goto Server > Create New
```JSON
ServerProperties{
    CoreDetails{
    "Server Name" : "minecraft",
    "Server Owner" : "boblukulus@outlook.com" //requires E-Mail
    },
    AllocationManagement {
        "Node" : "minecraft",
        "Default Allocation" : "192.168.65.161:25565"
    },
    ApplicationFeatureLimits {
        "Data Limit" : 0,
        "Allocation Limit": 0,
        "Backup Limit" : 3
    },
    RessourceManagement {
        "CPU Limit" : 0, //Unlimited
        "CPU Pinning" : NULL, //Empty
        "Memory" : 0, // Use Max set memory set for Node (4G)
        "Swap" : 0, //No Swap
        "Disk Space" : 0, //Max Space set for Node (8G)
        "Block IO Weight" : 500, //default (Dont know how to handle that otherwise)
        "OOM Killer" : False
    },
    NestConfiguration {
        "Nest" : "Minecraft",
        "Egg" : "Vanilla Minecraft",
        "Skip Egg Install Script" : False
    },
    DockerConfigutation {
        "Docker Image": "Java 21"
    }
    "Server Version": 
}
```
Create Server

Go to the icon in the tab bar on the most right (no tag)
Wait for Docker Container to Build (Took about 4min)
Accept the EULA with `I accept` button
Connect to server with minecraft the set IP

# Monitoring (23 Sept 2025)

## Install node exporter (Prometheus)

```bash
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-arm64.tar.gz # Download packages
tar xzf node_exporter-1.9.1.linux-arm64.tar.gz  # Extract tar package
sudo mv node_exporter-1.9.1.linux-arm64/node_exporter /usr/local/bin/
sudo useradd --no-create-home --shell /usr/sbin/nologin node_exporter || true # Create user for node explorer
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

Create SystemD Service for node explorer

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```systemd
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Enable Daemon

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

## Expose Minecraft metrics
! Works only on Vanilla or Fabric Minecraft Servers

### Enable RCON in Server Properties

In Pterodactyl go to the server > `Files` and open `server.properties`

Change the following lines to the shown values

```server.properties
enable-rcon=true
rcon.password=rconPW
rcon.port=25575
```

Click `Save Content` at the bottom

Restart the Server (Minecraft server)

### Run minecraft exporter Container

```bash
sudo docker run -d \
  --name=minecraft-exporter \
  -e RCON_HOST=127.0.0.1 \
  -e RCON_PORT=25575 \
  -e RCON_PASSWORD=rconPW \
  -p 9150:9150 \
  heathcliff26/minecraft-exporter:latest
```

<!-- Snapshot 3 -->

## Grafana + Prometheus install
### Prometheus

For security reasons, we'll create a dedicated user for Prometheus.

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

Download the latest Prometheus version, extract it, and move the files to the appropriate directories.

```bash
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-arm64.tar.gz
tar xvf prometheus-2.52.0.linux-arm64.tar.gz
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo mv prometheus-2.52.0.linux-arm64/prometheus /usr/local/bin/
sudo mv prometheus-2.52.0.linux-arm64/promtool /usr/local/bin/
sudo mv prometheus-2.52.0.linux-arm64/consoles /etc/prometheus
sudo mv prometheus-2.52.0.linux-arm64/console_libraries /etc/prometheus
sudo mv prometheus-2.52.0.linux-arm64/prometheus.yml /etc/prometheus/prometheus.yml
```

Set the ownership of the directories and files.

```bash
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
```

Now, we need to configure Prometheus to scrape metrics from our exporters. Open the Prometheus configuration file:

```bash
sudo nano /etc/prometheus/prometheus.yml
```

And replace the content with the following:

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name:   
    'prometheus'   
    static_configs:
      - targets: ['localhost:9090']
  
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'minecraft'
    static_configs:
      - targets: ['localhost:9150']
```

To run Prometheus as a service, create a systemd service file.

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Paste the following content into the file:

```systemd
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries
    
[Install]
WantedBy=multi-user.target
```

Reload systemd and start the Prometheus service.
    
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

You can check the status of the service with `sudo systemctl status prometheus`. Prometheus should now be running on port 9090.

For some reason (most definitely fucking Linux) I needed to wait for about 30s and restart the service#
```bash
sudo systemctl restart prometheus.service
sudo systemctl status prometheus.service
```

It should now be running

### Grafana

Install Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
```

Start Grafana

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
```
Again the status of the grafana-server can be checked with `systemctl status grafana-server.service` I highly recommend to check first if it is running. Because of linux

Grafana will be running on port 3000. You can access it by navigating to `http://192.168.65.161:3000`. The default login is `admin` with the password `admin`. You will be prompted to change the password after your first login.

I changed it to "changeme"

### Configure Grafana
1.  Log in to your Grafana dashboard.
2.  Click **Data Sources** on the left sidebar.
3.  Clic on **Add data source** and select (Double click) **Prometheus**.
4.  In the HTTP section, set the URL to `http://localhost:9090`.
5.  Click **Save & Test**. You should see a "Successfully queried the Prometheus API" message.


Import a Dashboard for Node Exporter


Grafana has a vast library of pre-built dashboards. We will import a popular one for the node exporter.

1.  On the left sidebar, click on **Dashboard** and select **Create dashboard**.
2. Click `Import dashbard`
2.  In the "Import via grafana.com" field, enter the dashboard ID `1860` and click **Load**.
3.  On the next screen, select your Prometheus data source and click **Import**.

### Troubleshooting 1
```bash
lf10b@main:~$ sudo ls -l /var/lib/pterodactyl/
total 32
drwx------ 2 root root  4096 Aug 28 16:44 archives
drwx------ 2 root root  4096 Sep 23 11:48 backups
-rw-r--r-- 1 root root    50 Sep 24 10:44 states.json
drwx------ 4 root root  4096 Aug 29 18:25 volumes
-rw-r--r-- 1 root root 16384 Sep 24 10:08 wings.db
lf10b@main:~$ sudo docker ps -a
CONTAINER ID   IMAGE                                    COMMAND                  CREATED          STATUS                    PORTS                                                              NAMES
6fef2fe7c63c   ghcr.io/pterodactyl/yolks:java_21        "/__cacert_entrypoin…"   36 minutes ago   Up 36 minutes             192.168.65.161:25565->25565/tcp, 192.168.65.161:25565->25565/udp   f2fdf9ed-7435-4d95-be54-bcbc32d94ad3
e94658b2f6a9   heathcliff26/minecraft-exporter:latest   "/minecraft-exporter"    23 hours ago     Exited (1) 23 hours ago                                                                      minecraft-exporter
```

Remove old exited docker container

```bash
sudo docker rm minecraft-exporter
```

```bash
sudo docker run -d \
    --name=minecraft-exporter \
    -e RCON_HOST=127.0.0.1 \
    -e RCON_PORT=25575 \
    -e RCON_PASSWORD=rconPW \
    -p 9150:9150 \
    -v /var/lib/pterodactyl/volumes/f2fdf9ed-7435-4d95-be54-bcbc32d94ad3/world:/world \
        heathcliff26/minecraft-exporter:latest
```
