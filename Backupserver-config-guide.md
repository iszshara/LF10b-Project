# VM config
6144MB RAM
64GB SSD
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
        "Full Name" : "LF10backup",
        "Servername" : "backup",
        "username" : "lf10backup",
        "password" : "lf10backup"
    },
    "Ubuntu Pro" : False,
    "OpenSSH Server" : True
}
```

# Set static IPs

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens160:
      dhcp4: no
      addresses:
        - 192.168.65.161/24
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    ens192:
      dhcp4: true
    ens224:
      dhcp4: true
```

```bash
sudo netplan apply
```

## System Update
nach reboot ausführen:
`sudo apt update && sudo apt upgrade -y`