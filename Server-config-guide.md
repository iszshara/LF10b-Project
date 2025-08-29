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

## System Update
nach reboot ausführen:
`sudo apt update && sudo apt upgrade -y`

# Pterodactyl Installation
<!-- Guide: https://pterodactyl.io/panel/1.0/getting_started.html -->
<!-- might be smart to run as root-user -->
## Dependency installation
```BASH
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

Backuo encryption key 
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
    "Application URL" : "http://192.168.65.160/panel",
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

Create Server

Go to the icon in the tab bar on the most right (no tag)
Wait for Docker Container to Build (Took about 4min)
Accept the EULA with `I accept` button
Connect to server with minecraft the set IP